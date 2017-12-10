---
title: Action cable in Rails 5
teaser: An introduction to websockets in Rails
category: [Frontend, Backend]
tags: [Rails]
---

One of the most interesting features that will be released in Rails 5 is action cable.
Action cable adds a very important capability to Rails as part of the framework which is WebSockets. In this post, I will explain the advantages of using WebSockets, when to use them and how.

## what are WebSockets?

WebSocket is a protocol providing full-duplex communication channels over a single TCP connection. It was invented in 2008, and it is supported by all new versions of the common web browsers. Full-duplex communication allows communication in both directions, from the server to the client and from the client to the server. Each side can initiate the communication. Full-duplex communication was widely spread long before the invention of WebSocket. It was used in telecommunication for many years and was adopted to computers to enable communication between rich client and server, PIR to PIR communication etc.
Before the invention of WebSocket, in the web, only the client could initiate the communication to the server. It sent a request to the server and got the response with the requested data. If the data needed to be refreshed, it had to be refreshed from the client, by a request from the user, or by a setTimeout method. The client was unaware of data changes that happened in the backend, and needed to check it all the time in order to be updated. It caused a lot of inconvenience especially in “real time” applications, where the user expected that the data will be updated.
WebSocket solves this problem. It allows the server to “notify” the client when the data changes. The client no longer has to check if something had been changed. It will get a message. It makes things a lot easier and accurate. Using WebSockets was available in older rails version by using external libraries. Action cable comes to simplify it and solve it the “Rails way”.

## Action cable
Action cable provides a JavaScript framework to the client and a ruby framework to the server.
Action cable can be implemented on two ways. As part of the application server, or by running in a separate server. Each server can handle multiple connection instances. It has one connection instance per WebSocket connection. A single user may have multiple WebSockets open to the server if they use multiple browser tabs or devices. The client of a WebSocket connection is called the consumer.
The client subscribe to a channel. The channel acts very similar to rails controller, it encapsulates its functionality with its methods. After the subscription, the channel will broadcast data to all the subscribers.

## Action cable in practice

I will create a “Google Docs” like application called “DOX”. This application allows multiple users to edit a document simultaneously. Each user will see immediately the changes that his friend did in the document.
The sources can be found - here.
So let's get started!
First we need to install Rails 5 (in the time of writing this post, it is rc1)
```bash
gem install rails --pre
```
Next I will create the application:
```bash
rails new Dox --database=postgresql
```
I will create a simple model for the document:
```bash
 rails g model document title:string body:string
```
And I will create a controller to handle the client’s requests.
```ruby
# app/controllers/document_controller.rb
class DocumentsController < ApplicationController

  def index
  end

  def docs
    render json: Document.all
  end

  def show
    render json: Document.find(params[:id])
  end

  def create
    render json: Document.create!(title: params[:title])
  end

  End
```
I’m using Angular as my frontend framework, so all responses are JSON objects.

Next,  I will add the support for my WebSockets:
I will create a Document channel:
``` ruby
# app/channels/documents_channel.rb
class DocumentsChannel < ApplicationCable::Channel
  def subscribed
      stream_from 'documents'
    end
end
```
This channel inherits the base ApplicationCable::Channel. In my case I will implement only the subscribe method. This method is responsible for subscribing to and streaming messages that are broadcast to this channel. Other popular methods are unsubscribe, for cleaning up, appear, to notify that there is a client and away  to notify that there is no client.
I want to broadcast to all subscribers when the document is being edited over the channel. I will add it to the document_controller:
``` ruby
def edit
    doc = Document.find(params[:id])
    doc.update!(body: params[:body]);
    ActionCable.server.broadcast 'documents',
        document: doc.body
  end
```
The edit method will update the document in the database and will notify all with a message that contains the updated data.

The easiest way to pass the address of the action cable server to the client would be adding  
``` html
  <%= action_cable_meta_tag %>
```
To  the head of application.html.erb. It will take the value from

``` ruby
Rails.application.configure do
…
config.action_cable.url = "ws://localhost:3000/cable"
end
```
And use it.
We will also have to define in  routes.rb:

``` ruby
mount ActionCable.server => '/cable'
```
Another option is to create the consumer  with the address of the server:
```javascript
App.cable = ActionCable.createConsumer("ws://example.com:28080")
App.cable = ActionCable.createConsumer("/cable")
```
Now I will implement the client side.
I am adding it to my controller init method:
``` js
this.App = {};

App.cable = ActionCable.createConsumer(); // use action_cable_meta_tag

App.documents = App.cable.subscriptions.create('DocumentsChannel', {
  received: function (data) {
    $scope.editedDoc.body = data.document;
  },
});
```
I am creating the consumer, subscribe on it with a function. The callback will refresh my data.
One more thing is to add cable to the application.js:
``` js
//= require cable
```
That's all, now every change will update each client.

## Publish subscribe mechanism
Action Cable uses adapters to subscribe and send messages. Few are included, such as Redis, Postgresql, async etc. We can also implement our own adapter. The popular for production is Redis. When the Action Cable server broadcast, it uses Redis channel maintained by Redis. The subscribed method of the channel is streaming messages sent over the  channel maintained by Redis.
Thus, Redis acts as a data store and ensures that messages will remain in sync across instances of our application.
Action Cable will look for the configuration in Rails.root.join('config/cable.yml'). When we generated our new Rails 5 app, we also generated a file, config/cable.yml, that looks like this:
``` ruby
production:
  adapter: redis
  url: redis://localhost:6379/1

development:
  adapter: async

test:
  adapter: async

```
## Deployment
Action Cable is powered by a combination of WebSockets and threads. All of the connection management is handled internally by utilizing Ruby’s native thread support, which means you can use all your regular Rails models with no problems as long as you haven’t committed any thread-safety sins.

The Action Cable server does not need to be a multi-threaded application server. This is because Action Cable uses the Rack socket hijacking API to take over control of connections from the application server. Action Cable then manages connections internally, in a multithreaded manner, regardless of whether the application server is multi-threaded or not. So Action Cable works with all the popular application servers -- Unicorn, Puma and Passenger.

Action Cable does not work with WEBrick, because WEBrick does not support the Rack socket hijacking API.

## Conclusion
WebSockets are great because it allows the client to be updated any time the data is changed. Action Cable makes thing very simple and adds a great force to rails. There are cons for using it such as performance issues, power (in mobile) and more, but I believe that WebSockets and Action Cable will be widely used because today there is a demand for “real time” web, and it is the best solution for it.

References:
- https://en.wikipedia.org/wiki/WebSocket
- https://github.com/rails/rails/tree/master/actioncable
- https://blog.heroku.com/archives/2016/5/9/real_time_rails_implementing_websockets_in_rails_5_with_action_cable