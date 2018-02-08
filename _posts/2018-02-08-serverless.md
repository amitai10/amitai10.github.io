---
title: Serverless
teaser: Serverless architecture - The future is here!
category: [Serverless]
tags: [Serverless]
---

In the end, as a developer I just want my code to run. I do not want to mess with servers, VM’s, dockers, containers, memory, CPU’s, network, load balancing, disk space... you get the point.  
Serverless architecture aims to do that for me.  
No more servers!

## What is Serverless?
A decade or more ago, if we wanted to deploy our applications, we needed to build data centers, with servers racks, freezing air condition and IT department that was busy all the time.  
After the invention of AWS, we had a much easier way to bootstrap and run our application using Amazon's infrastructure. It is called __IaaS__ (infrastructure as a Service).  
We could use their server and only pay money for using them. But we still need to configure and install everything ourselves.  
Then services like Heroku appeared, they gave us an abstraction of the servers that known as __PaaS__ (platform as a service).  
We didn’t have to do much, just deploy our code to Heroku and it worked. The problem was that it worked very good for monolithic applications, but for a complicated applications that use microservices it was harder.  
Then Amazon invented __FaaS__ (Function as a service). You do not need to deploy your application, only your functions. Amazon will take care of everything for you.  
FaaS is a synonym for serverless (in the rest of this post I will use both terms for serverless). It is not that the code is not running on a server, it is just that you do not care about it.

> It worth mentioning another approach of BaaS (backend as a service) that enables web or mobile applications to have a backend with some services such as authentication, database or storage, but didn’t allow them to run logic on the server (all the logic was on the client).

The are three major serverless providers: AWS Lambda, Google Cloud functions, and Microsoft Azure functions. Each of them is very good and selecting between them is a matter of convenience or personal preference.

## Serverless pros
Serverless or FaaS has many advantages:
- Stability - you do not need to monitor if your server is up. It is. Your provider is taking care of it.
- Scaling - no need to worry about it as well. If your application needs more resources, your provider will provide it without any action from you.
- Price - this is a tough one, but in the end, you do not need to spend money on standing still server, you pay only when you use it. In addition, you can save the salaries of IT and devops to support a complex solution.
- Deployment - you deploy your functions. No need to mess with installations, docker, containers, SSL, load balancers. Firebase gives a very smooth deployment process through its Firebase CLI. AWS Lambda has also tools that facilitates the process such as [Apex](http://apex.run/) or [serverlees.com](https://serverless.com/).

## Serverless cons
- Vendor lock-in -  When using serverless you are pretty much locked to the vendor. It is not likely to change the vendor because you use its special API’s, services such as database or storage.
- Bootstrap latency - it takes time for the function to start running. In the contrast to a regular server that is up and running, the function is not. So it takes time for it to be loaded and run.
- Stateless - I am not sure if this is a disadvantage or not because I do not like to use state on my server, but in FaaS, you simply cannot. 
- Server optimization - because we have no control of the server, we cannot hack it as we please, and we cannot make optimizations for our app.

## Event-driven programming
FaaS has a different approach from traditional server programming. Instead of a synchronous chain of procedures, FaaS is event-driven. A function is being called after an event is triggered. The event can be an Http request, file upload to storage, database change and more.
It takes time to get used to it, and design the app accordingly.
When writing app with Firebase, simple CRUD operations are trivial, and the functions give the added functionality such as sending a push notification, update a different record in the database etc.  

## Conclusion
As I can see it, Serverless is the future. You will need a very strong argument to deploy in any other way. It saves salaries of IT and devops, it is stable, scalable and reliable. Currently, there are still some drawbacks: slow bootstrap and performance, but I think that over time when the technology will become mature it will improve.

## References
https://martinfowler.com/articles/serverless.html
