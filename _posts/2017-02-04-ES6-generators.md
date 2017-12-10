---
title: ES6 generators
teaser: What are generators and how to use them in the real world
category: [JavaScript]
tags: [JavaScript]
---

One of the more complicated notions presented in ES6 is generators. This is a feature with a lot of power that enhance the JavaScript language, especially in areas such as asynchronous programing.

In this blog post I will describe what are generators and how to use them in the real world.

## Iterator
In order to understand generators, we need first to learn what is an iterator.

Iterator is a design pattern to go over elements in a collection without knowing nothing about the collection structure and the elements themselves.

Iterator has two methods:

- next() - return the next item of the collection
- hasNext() - returns true or false whether the collection has next item.

A simple implementation an iterator can be:
```js
function Iterator(collection) {
    let index = 0;
    return {
        next: () => {
            return collection[index++];
        },
        hasNext: () => {
            return index < collection.length;
        }
    }
}

const iter = Iterator([1,2,3]);
while (iter.hasNext()) {
    console.log(iter.next()); // 1 2 3
}

```

## for..of
Another cool feature of ES6 is for..of loop. This operator creates loop with iterators objects. The Iterator object is a JavaScript object that implement the _iterator_ JavaScript protocol. The protocol defines how to implement the iterator design pattern. The object has a _next_ method that return object with two keys:
- value
- done (boolean)

JavaScript objects such as Array, Map and Set implements this protocol. If we want to do our simple implementation to our iterator object, it can be:

```js
function Iterator(collection) {
    let index = 0;
    const iter = {};
    iter[Symbol.iterator] = () => {
        return {
            next: () => {
                if (index < collection.length) {
                    return {
                        value: collection[index++],
                        done: false
                    };
                } else {
                    return {
                        value: null,
                        done: true
                    };
                }
            }
        }
    }

    return iter;
}

const iter = Iterator([2,21,33]);
for (let x of iter) {
    console.log(x); // 2 21 33
}
```
The for..of operator looks for the __Symbol.iterator__ of the object and uses it to iterate.

## Generators
Generators come to make the creation of generators easier. Instead of all the boilerplate code above, we can use them:
```js
function* Iterator(collection) {
    let index = 0;
    while (index < collection.length) {
        yield collection[index++];
    }
}

const iter = Iterator([22,29,35]);

for (let x of iter) {
    console.log(x); // 22 29 35
}
```
As we can see the implementation is much cleaner with much less code.   
But, what’s going on here?  
Let's start of defining generators:
> A generator is a special type of function that works as a factory for iterators. A function becomes a generator if it contains one or more yield expressions and if it uses the function* syntax.

This is a very simple generator:
```js
function* simpleGenerator(){
    yield 'a';
    yield 'b';
    yield 'c';
}

const iter = simpleGenerator();

console.log(iter.next()); // { value: 'a', done: false }
console.log(iter.next()); // { value: 'b', done: false }
console.log(iter.next()); // { value: 'c', done: false }
console.log(iter.next()); // { value: undefined, done: true }
```
Its definition contains ‘function*’, and inside it has __yield__ keyword. The generator manages its state and return upon any call the next value. Each returned value has two elements: _value_, and _done_. As we can see, as long as it has next value it returns it until it doesn’t and then it returns _done: true_.

## Bidirectional communication
Generators can receive parameters and use them:
```js
function* foo(x) {
   const y = yield x;
   const z = yield y;
   return z;
}

const gen = foo(1);

console.log(gen.next());    // { value: 1, done: false }
console.log(gen.next(2));   // { value: 2, done: false }
console.log(gen.next(3));   // { value: 3, done: true }
```
The first _yield_ returns the first parameter that the generator receives in its creation (x), then _y_ gets the value from the _next()_ methods, and so does _z_. The yield plays two roles, return the next element to the caller, and receive parameter to the generator.  As you can see, I returned z, instead of “yielding”. The result was that the _done_ is true and the generator finished it elements.

## Generators and Asynchronous
One of the great feature of generator is that they are asynchronous. When we use the _yield_ keyword, it waits until it receives a value and then continues. It allows us to do async tasks (like fetching data from storage, ajax calls…).  
For example:
```js
function asyncCall(x) {
    setTimeout(() => {
        gen.next(x + 5)
    }, 1000);
}

function* asyncGenerator() {
    let val = yield asyncCall(5)
    console.log('Value =', val);    // Value = 10
    val = yield asyncCall(10)
    console.log('Value =', val);    // Value = 15
}

const gen = asyncGenerator();
gen.next();
```
I use the generator object (gen) to return the value inside the generator. Imagine that instead of using _setTimeout_, it was _fetch_ request.
But, there something awkward here: we use the generator inside the asyncCall, and it is very inconvenient and weird. We will see how we can improve it with the use of promises.

## Generators with promises
When we think about ES6 and async programing, we think about promises. The combination between them is very powerful and allows to avoid the horrible callback hell. You can refresh your memory about promises in my previous [blog post](https://www.spectory.com/blog/Asyncronous%20Client-Side%20Javascript%20Primer).
It will look like this (inspired by [Kyle Simpson greate article](https://davidwalsh.name/async-generators)):
```js
function asyncCall(x) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(x + 5)
        }, 1000);
    });
}

function runGenerator(generator) {
    const gen = generator();
    let ret;
    (function iterate(val){
        ret = gen.next(val);
        if (!ret.done) {
            ret.value.then(iterate);
        }
    })();
}

runGenerator(function* asyncGenerator() {
    let val = yield asyncCall(5)
    console.log('Value =', val);    // Value = 10
    val = yield asyncCall(10)
    console.log('Value =', val);    // Value = 15
});
```
The first thing to notice is that we took the generator out of the asyncCall and changed to return a promise. The next thing is the use of helper method that deals with the promise (runGenerator). This methods gets a generator as parameter, activate it and handle it’s returned values. Take a few minutes to understand this implementation because it is not that obvious. Of course this is a very naive implementation that does not deal with errors, and other stuff. Thankfully there are bunch of open source libraries (co, Q, bluebird) that implement it and we can use them instead of this.

## Introducing co.js
Co is generator based control flow goodness for nodejs and the browser, using promises, letting you write non-blocking code in a nice-ish way.
The best way to demonstrate it is:
```js
var co = require('co');

co(function *(){
  // yield any promise
  var result = yield Promise.resolve('co');
  console.log('hello', result); //hello co
});
```
Co gets generator (as our runGenerator) and execute it.
In order to use it in our previous example:
```js
var co = require('co');

function asyncCall(x) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(x + 5)
        }, 1000);
    });
}

co(function* asyncGenerator() {
    let val = yield asyncCall(5)
    console.log('Value =', val);    // Value = 10
    val = yield asyncCall(10)
    console.log('Value =', val);    // Value = 15
});

```
Looks much cleaner. We can use this method to do async programing calls as it was synchronous.

## Async/Await
ES7 Async/Await take this methodology to the next level and even conceal the use of the generator based control flow.
The code simply looks like this:
```js
function asyncCall(x) {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            resolve(x + 5)
        }, 1000);
    });
}

(async function asyncGenerator() {
    let val = await asyncCall(5)
    console.log('Value =', val);    // Value = 10
    val = await asyncCall(10)
    console.log('Value =', val);    // Value = 15
})();
```
We simply replaced the _function*_ with _async function_ and _yield_ with _await_.
There is a lot of resemble async/await to generators, and this is not a wonder that babel transpiler uses generators in order to implement async/await.

## References
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_Generators
- https://davidwalsh.name/es6-generators
- [ES2015 Iterators and Generators - Dan Shappir](https://www.youtube.com/watch?v=mM7k1OuhK5E)
- https://github.com/tj/co

