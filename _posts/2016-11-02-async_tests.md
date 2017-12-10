---
title: Async tests with mocha in node
teaser: How to ease this pain with the new Async/Await feature of ES7
category: [JavaScript]
tags: [JavaScript]
---

Node JS is Asynchronous. It's a pain. It’s a pain to developing this way and it’s a pain writing tests for this framework.

In this blog post I will show how to ease this pain with the new Async/Await feature of ES7.

I love writing unit tests, it makes me develop faster and better.  
When I test my code, I use it against the database. Some will argue that this is not unit testing, But I find this method to be easier with a lot of advantages, although it takes longer to run my tests.  
Unit testing against the database in Ruby or Java is like testing against mocks and stubs. In node it is different because every call to the database is asynchronous.

In this example I’m using mocha with chai. The ORM I’m using is sequelize.

## The data model 

My data model will be Company with employees, and every employee will has an address.
```JavaScript
  var Company = sequelize.define('Company', {
    name: DataTypes.STRING
  }, {
    classMethods: {
      associate: function(models) {
        Company.hasMany(models.Employee);
      }
    }
  });


 var Employee = sequelize.define('Employee', {
    name: DataTypes.STRING
  }, {
    classMethods: {
      associate: function(models) {
        Employee.hasOne(models.Address);
      }
    }
  });
  return Employee;


 var Address = sequelize.define('Address', {
    city: DataTypes.STRING,
    street: DataTypes.STRING,
    house: DataTypes.INTEGER
  });
```

## Using ES5
In my test, I would like to test that the data model is correct,  with all the associations.  
This is how it writen in ES5:
```JavaScript
describe('company model', function () {
  beforeEach((done) => {
    db.sequelize.sync({ force: true }) // drops table and re-creates it
      .then(() => {
        db.Company.create({ name: 'Soectory' })
        .then(function (company) {
          db.Employee.create({ name: 'Amitai Barnea' })
          .then(function (employee) {
            company.addEmployee(employee);
            db.Address.create({ city: 'Netanya' })
            .then(function (address) {
              employee.setAddress(address)
              .then(done());
            });
          });
        });
      })
      .catch((error) => {
        done(error);
      });
  });


  it('should have all associations', function (done) {
    db.Company.findOne({id: 1 })
    .then(function (company) {
      company.getEmployees()
      .then(function (employees) {
        expect(employees.length).to.equal(1);
        expect(employees[0].get().name).to.equal('Amitai Barnea');
         employees[0].getAddress()
         .then( function (address) {
           expect(address.get().city).to.equal('Netanya');
	done();
         });
      });
    });
  });
});

```
Oohh!! This looks awful. This is so hard to read or maintain, it is hard to know, when each function starts or ends.  
It is callback hell with promises!

## using ES7 Async/Await

There got to be a better way, right?  
Yes, ES7 async/await (it is a new feature of node 7) comes to help us.  
This is how it looks in ES7:
```JavaScript
describe('company model', () => {
  beforeEach((done) => {
    db.sequelize.sync({ force: true }) // drops table and re-creates it
      .then(async () => {
        const company = await db.Company.create({ name: 'Spectory' });
        const employee = await db.Employee.create({ name: 'Amitai Barnea' });
        await company.addEmployee(employee);
        const address1 = await db.Address.create({ city: 'Netanya' });
        await employee.setAddress(address1);
        done();
      })
      .catch((error) => {
        done(error);
      });
  });


  it('should have all associations', async () => {
    const company = await db.Company.findOne({id: 1 });
    const employees = await company.getEmployees();
    expect(employees.length).to.equal(1);
    expect(employees[0].get().name).to.equal('Amitai Barnea');
    const address = await employees[0].getAddress();
    expect(address.get().city).to.equal('Netanya');
  });
});
```
So much cleaner, I can read it, I can add more scenarios. It make sense.
Things to notice:
- async  - at the main functions.
- await for every async call.
- I’m not using done in the test.


## How to activate mocha with ES7? 

Super easy just add babel:
```
./node_modules/.bin/mocha --compilers js:babel-core/register test/server -w --recursive
```

## Conclusion

Unit testing is fundamental in agile deployment.  
They are becoming more and more crucial for the deployment because we use methods like CI/CD.  
We want to make sure that the application we are deploying is not broken. We prefer using automated tests rather than human effort.  
Although all developers know that, a lot of them tend to skip them or to use them less then they should.  
Today we have great tools for unit testing, and this area is evolving fast.


Use these tools and Test!

