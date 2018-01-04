---
title: A deep dive deep into TypeScript decorators
teaser: What are TypeScript decorators and how to use them
category: [JavaScript]
tags: [JavaScript]
---

Almost twenty years ago, I hiked with some friends, I was then in my BSC's in the Technion. One of the guys was a Ph.D. student and I asked him what’s hot in computer science. He replied that they are investigating AOP (aspect-oriented programing). That was the first time I heard of it. Since then, I used it from time to time (especially with Java and .Net).    
When I started using typescript I came across AOP again.  
What is AOP?  
AOP according to [wikipedia](https://en.wikipedia.org/wiki/Aspect-oriented_programming) AOP is a programming paradigm that aims to increase modularity by allowing the separation of cross-cutting concerns. It does so by adding additional behavior to existing code (an advice) without modifying the code itself. 

The biggest advantage of using AOP is the separation of irrelevant code from the main functionality of the method to a different place. The best example is logging or authorization. Instead of doing it inside the function which makes it clunky, we pull it out and only declare that the function is doing it. In addition,  we can change it in a single place rather than in all the code base.

```ts
// without AOP:
function add(x, y) {
  log(‘foo was called!’);
  If (!validate(arguments)) {
      throw(...)
  }
  If (!authorized()) {
      throw()
  }
  return x + y;
}

// with AOP
@log
@validate
@authorize
function add(x, y) {
  return x + y;
}
```
The main purpose of the method becomes very clear and much more testable.

So, how can I use this paradigm in typescript?  
By using decorators!  
Decorators provide a way to add both annotations and a meta-programming syntax for class declarations and members.  
Decorators are a stage 2 proposal for JavaScript and are available as an experimental feature of TypeScript.

## TypeScript Decorators

Decorators can be attached to:
- Methods
- Classes
- Properties
- Parameters
- Accessor

It is very easy to created and attach decorator and make your code cleaner and maintainable (just like AOP suggests).

In This blog post, I will describe how to do so for every kind.

## Method decorator
Let’s take the previous example from above and implement it with decorators:  
First I will create the decorator function, and then I will attach it to the method:
```ts
function log(target, key, descriptor) {
    console.log(`${key} was called!`);
}

class P {
  @log
  foo() {
      console.log(‘Do something’);
  }
}
const p = new P();
p.foo();

// printed to console :
// foo was called!
// Do something

```
Let's go over the code above:
```ts
function log(target, key, descriptor) {
...
}
```
In order to create a decorator, we just have to create a simple function. The function receives three parameters:
- target - Either the constructor function of the class for a static member or the prototype of the class for an instance member.
- key - The name of the member.
- descriptor - The Property Descriptor for the member.

```ts
@log
foo() {
  ...
}
```
We simply add @function_name above the method, and that’s it, typescript does the rest for us.
The decorator function will be called every time the method is called.

Let’s do something more advanced such as logging the method’s parameters:
```ts
function log(target, key, descriptor) {
  const originalMethod = descriptor.value;
  descriptor.value = function () {
    console.log(`${key} was called with:`,arguments);
    var result = originalMethod.apply(this, arguments);
    return result;
  };
  return descriptor;
}

class P2 {
  @log
  foo(a, b) {
    console.log(`Do something`);
  }
}
const p2 = new P2();
p2.foo('hello', 'world');
// printed to console :
// foo was called with: { '0': 'hello', '1': 'world' }
// Do something
```
Here, I took advantage of the third argument, the descriptor. I replaced it with a new functionality, in which I print the value of the arguments and then run the original one!

As you can see, the options are endless. 

## Class decorator
A Class Decorator is declared just before a class declaration. The class decorator is applied to the constructor of the class and can be used to observe, modify, or replace a class definition. If the class decorator returns a value, it will replace the class declaration with the provided constructor function.

For example, here I extend my class to support new methods and properties
```ts
function init(target) {
  return class extends target {
    firstName = "Amitai";
    lastName = "Barnea";
    sayMyName() {
      return `${this.firstName} ${this.lastName}`
    }
  }
}
@init
class P {
  name: string;
  constructor() {
  }
}
let p = new P();
console.log(p.sayMyName()); // Amitai Barnea
``` 
Angular2+ does an extensive use of class decorator with @component and @ngModule. 

## Property Decorator
A property decorator is declared just before a property declaration. The expression for the property decorator will be called with the prototype of the class and the name of the member.
In The following example, I will change the setter and the getter of the property. I will demonstrate how to set other fields in the object.

```js
function calcCircleParams(target: any, key: string) {
  // Property value.
  let _val = this[key];
  // Property getter.
  const getter = function () {
    return _val;
  };
  // Property setter.
  const setter = function (newVal) {
    _val = newVal;
    this['Area'] = _val*_val*Math.PI;
    this['Circumference'] = 2*_val*Math.PI;
  };
  // Delete property.
  if (delete this[key]) {
    // Create new property with getter and setter
    Object.defineProperty(target, key, {
      get: getter,
      set: setter,
      enumerable: true,
      configurable: true 
    });
  }
}

class Circle {
  @calcCircleParams
  public Radius: Number;
  public Area: Number;
  public Circumference: Number;
  constructor() {
  }
}
let c = new Circle();
c.Radius = 3;
console.log(`Radius: ${c.Radius}, Area: ${c.Area}, Circumference: ${c.Circumference}`); // Radius: 3, Area: 28.274333882308138, Circumference: 18.84955592153876
c.Radius = 5;
console.log(`Radius: ${c.Radius}, Area: ${c.Area}, Circumference: ${c.Circumference}`); // Radius: 5, Area: 78.53981633974483, Circumference: 31.41592653589793
```

I delete the original behavior and set a new one. In the setter, the Area and the circumference are calculated according to the radius of the circle.

## Parameter Decorator
A Parameter Decorator is declared just before a parameter declaration. The parameter decorator is applied to the function for a class constructor or method declaration. The expression for the parameter decorator will be called as a function at runtime, with the following three arguments: the prototype of the class, the name of the member, and the ordinal index of the parameter in the function’s parameter list.

In the next example I will use parameter decorator and method parameter together:
```js
function required(target: any, key: string, index: number) {
  var metadataKey = `__required_${key}_parameters`;
  if (Array.isArray(target[metadataKey])) {
    target[metadataKey].push(index);
  }
  else {
    target[metadataKey] = [index];
  }
}
function validate(target, key, descriptor) {
  var originalMethod = descriptor.value;
  descriptor.value = function (...args: any[]) {
    var metadataKey = `__required_${key}_parameters`;
    var indices = target[metadataKey];
    for (var i = 0; i < args.length; i++) {
       if (arguments[i] == undefined) {
         throw 'missing required parameter'
       }
    }
    var result = originalMethod.apply(this, args);
    return result;
  }
  return descriptor;
}
class Calculator {
  @validate
  add(@required a: number, @required b: number) {
    return a + b;
  }
}
const c = new Calculator();
console.log(`result is: ${c.add(2,3)}`);     // result is: 5
console.log(`result is: ${c.add(2,undefined)}`);    // throw 'missing required parameter'
```
Let’s go over the code in order to understand it:
I declare class calculator with method ‘add’ that decorated with ‘validate’, and params that are decorated with ‘required’.
It means that every time the method is called, the ‘validate’ decorator will be called before. The parameter decorator is called before the method parameter and inserts the required fields into ‘metadatKey’ array. The ‘validate’ decorator goes over all the fields in the ‘metadatKey’ field and verifies that they are defined. If not, it throws an exception.

## Additions
There are more things that you can do with decorators that I didn’t describe here such as compose decorators, accessor Decorators, limitation and more.
You are welcome to read about all of that in the official typescript [documentation](https://www.typescriptlang.org/docs/handbook/decorators.html).

## Conclusion
Decorators can help writing better code, clear and reasonable:
- Separate the essence of the class/method. 
- Describe the object, for instance, a class with ‘@component’ decorator is immediately recognized as one, much more fast then if it would have certain properties or methods within.
- Create common code to reuse in many places, such as logging, validation etc.

I recommend using them!

## References
- https://www.typescriptlang.org/docs/handbook/decorators.html
- https://gist.github.com/remojansen/16c661a7afd68e22ac6e

