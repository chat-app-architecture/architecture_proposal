# Chat API Architecture Proposal

This is a Systems Design solution for a Chat app. I present a High Level and Low Level overview and discuss some of the main critical parts of a chat application.

- [Systems Design - High Level](#systems-design---high-level)
  - [Gathering System Requirements](#gathering-system-requirements)
  - [Coming Up With A Plan](#coming-up-with-a-plan)
  - [Persistent Storage Solution and App Load](#persistent-storage-solution-and-app-load)
  - [Load Balancing](#load-balancing)
  - [Sharding](#sharding)
  - [Pub-Sub System for Real-Time Behavior](#pub-sub-system-for-real-time-behavior)
- [Systems Design - Low Level](#systems-design---low-level)
  - [Gathering System Requirements](#gathering-system-requirements)
  - [Actions](#actions)
  - [Coming Up With A Plan](#coming-up-with-a-plan)
  - [Node.js + Redis + Socket.io](#nodejs---redis---socketio)
    - [How it works](#how-it-works)
  - [C4](#c4)
    - [System Context](#system-context)
    - [Container Diagram](#container-diagram)
    - [Component Diagram](#component-diagram)
      - [Api engine](#api-engine)
      - [Authentication engine](#authentication-engine)
    - [Code](#code)
    - [Rails API](#rails-api)
      - [Project Structure](#project-structure)
        - [Databases](#databases)
        - [Asynchronous jobs](#asynchronous-jobs)
        - [Error Handling](#error-handling)
        - [Controllers](#controllers)
        - [Services](#services)
        - [Documentation of API](#documentation-of-api)
        - [Rollbar](#rollbar)
        - [Circle CI](#circle-ci)
        - [Ruby Scanners](#ruby-scanners)
      - [Tests and Delivery Automation](#tests-and-delivery-automation)
      - [Docker](#docker)

# Systems Design - High Level

## Gathering System Requirements

We're designing the core communication system behind the app, which allows users to send instant messages in chat groups.

Specifically, we'll want to support:
Loading the most recent messages in a group when a user clicks on the group.
Sending and receiving messages instantly, in real time.

The system should have low latencies and high availability, catering to a few regions of roughly 20 million users. The largest groups will have as many as 50,000 users.

That being said, for the purpose of this design, we should primarily focus on latency and core functionality.

## Coming Up With A Plan

We'll tackle this system by dividing it into two main sections:
Handling what happens when a Chat app loads.
Handling real-time messaging.

We can further divide the first section as follows:
Seeing all of the groups that a User is a part of.
Seeing messages in a particular group.

## Persistent Storage Solution and App Load

While a large component of our design involves real-time communication, another large part of it involves retrieving data (groups, messages, etc.) at any given time when the Chat app loads. To support this, we'll need a persistent storage solution.

Specifically, we'll opt for a SQL database since we can expect this data to be structured and to be queried frequently.

We can start with a simple table that'll store every Chat app group.

### Groups

```ruby
{
  "id": uuid,
  "name": string,
  "description": string
}
```

Then, think about the Users table.

### Users

```ruby
{
  "id": uuid,
  "name": string,
  "mobile_number": string
}
```

Then, we can have another simple table representing group-user pairs: each row in this table will correspond to a particular user who is in a particular group.

### Group Users

```ruby
{
   "id": uuid,
   "name": string,
   "description": string,
   "group_id": uuid,
   "user_id": uuid
}
```

We'll naturally need a table to store all historical messages sent on the Chat app. This will be our largest table, and it'll be queried every time a user fetches messages in a particular group. The API endpoint that'll interact with this table will return a paginated response, since we'll typically only want the 50 or 100 most recent messages per group.

Also, this table will only be queried when a user clicks on a group; we don't want to fetch messages for all of a user's groups on app load, since users will likely never look at most of their groups.

### Group Messages

```ruby
{
  "id": uuid,
  "group_id": uuid,
  "sender_id": uuid,
  "body": string
}
```

## Load Balancing

For all of the API calls that clients will issue on app load, including writes to our database, we're going to want to load balance.

For the sake of the MVP/Alpha version, we can have a simple round-robin load balancer, forwarding requests to a set of auto scalable server clusters that will then handle passing requests to our database.

## Sharding

Since our tables will be very large, especially the historical messages table, we'll need to have some sharding in place.

The natural approach is to shard based on group size: we can have the biggest groups in their individual shards, and we can have smaller groups grouped together in other shards.

## Pub-Sub System for Real-Time Behavior

With hundreds of messages placed every second, the messages table will be pretty massive. We'll need to figure out a robust way to actually send messages and to update our table.

We can design this part of our system with a Publish/Subscribe pattern. The idea is to use a message queue like Apache Kafka or Google Cloud Pub/Sub and to have a set of topics that user ids map to. This gives us at-least-once delivery semantics to make sure we don't miss new messages. When a user sends a message, the API server writes a row to the database and also creates a message that gets routed to a topic for that user (using hashing), notifying the topic's subscriber that there's a new message.

This gives us a guarantee that for a single user, we only have a single thread trying to send their messages at any time.

In order to have a more scalable solution that could support millions of simultaneous requests, subscribers of topics can be rings of 3 workers (clusters of servers, essentially) that use leader election to have 1 master worker do the work for the cluster (this is for our system's high availability) -- the leader grabs messages as they get pushed to the topic and executes the messages for the Users contained in the messages by calling the exchange. As mentioned above, a single Users' messages are only ever handled by the same cluster of workers, which makes our logic and our SQL queries cleaner.

As far as how many topics and clusters of workers we'll need, we can do some rough estimation. If we plan to send millions of messages per day, that comes down to about 10-100 messages per second. If we assume that the core execution logic lasts about a second, then we should have roughly 10-100 topics and clusters of workers to process messages in parallel.

```
 ~100,000 seconds per day (3600 * 24)
  ~1,000,000 messages per day
  messages bunched in 1/3rd of the day
  --> (1,000,000 / 100,000) * 3 = ~30 messages per second
```

# Systems Design - Low Level

## Gathering System Requirements

We're designing the core communication system behind the app, which allows users to send instant messages in chat groups.

Specifically, we'll want to support:
Basic authentication for users
Loading the most recent messages in a group when a user clicks on the group.
Sending and receiving messages instantly, in real time.

### Actions

## Coming Up With A Plan

We'll tackle this system by dividing it into two main sections:
Handling what happens when a Chat app loads.
Handling real-time messaging.

We can further divide the first sections as follows:

## Nodejs - Redis - Socketio

The infrastructure is easy to understand: the Node server subscribes to Redis events which are published by Rails server. Clients (browsers or mobile apps) are connected to the Node server, and receive events through that connection. In other words, it contains node server functionality which subscribes to channels on redis and emits messages using Socket.io library.

### How it works

When a User creates a new message, the Rails API publishes that new message to Redis on the “routing” channel.
On the Node server, each client subscribing to “routing” receives that new message.
The message is pushed to the client using Socket.IO.
Within the browser, Socket.IO receives that new message and “publishes” that change to the frontend application.
The frontend application listens for changes to messages and adds the new message to itself.

The main advantage of this approach is that if the Node server crashes, your application will work as it always has (without real-time).

## C4

Below I’m using the C4 methodology (System Context, Container Diagram, Component Diagram and Code) to represent my system in a high and low level perspective.

### System Context

<img src="https://github.com/chat-app-architecture/architecture_proposal/blob/main/images/system-context.png" width=500 />

* Note that I didn’t include any database replica or sharding logic to the Rails API because it would make it more complex.

### Container Diagram

The Rails API explores the concepts of Rails engines in order to make sure our system is modular and we can reuse the chat engine in other Rails applications. We treat each engine as an isolated component inside this backend solution as they have a single responsibility:

- `Authentication Engine`: Handles the Authentication for Users using devise and devise-jwt.

- `Api Engine`: Receives all the HTTP calls and interacts with the other components. Parses all the responses and send as serializers.

- `Chat Engine`: Component that contains business logic regarding the chat component such as Groups, GroupMessage. It offers models and factories.

<img src="https://github.com/chat-app-architecture/architecture_proposal/blob/main/images/tree-folder.png" width=300 />

As you can see, this is not a normal Rails API where you’d see the standard MVC at the top of the tree folder.

When we are building a car from scratch, we have to think about which engines a car is going to have. Backend APIs are not different. You should componentize your apps.

### Component Diagram

#### Api engine

The first version of the Chat engine contains GroupMessagesController and GroupsController. These controllers provide Controller actions to manage groups and messages.

<img src="https://github.com/chat-app-architecture/architecture_proposal/blob/main/images/api-engine.png" width=500 />

#### Authentication engine

The first version of the Authentication Engine uses devise and devise-jwt gems. Devise is a flexible authentication solution for Rails based on Warden. The features that we can take advantage are:

- `Database Authenticatable`: hashes and stores a password in the database to validate the authenticity of a user while signing in. The authentication can be done both through POST requests or HTTP Basic Authentication.
- `Registerable`: handles signing up users through a registration process.

<img src="https://github.com/chat-app-architecture/architecture_proposal/blob/main/images/authentication-engine.png" width=500 />

### Code

The user_resources table is necessary because we can’t create relationships between the users table and the group_messages table since they live in different Rails engines. We should treat these components as they were different systems in different servers (IPs) that would require HTTP calls to be accessed.

<img src="https://github.com/chat-app-architecture/architecture_proposal/blob/main/images/database.png" width=500 />

## Rails API

### Project Structure

#### Databases

Our first option is to use PostgreSQL as a transactional database, and Redis to deal with Sidekiq queues and cache. We can think about adding RabbitMQ or Google Cloud Pub/Sub as the project evolves.

We'll rely on cache structures and indexing of Postgres to deal with the amount of messages. PostgreSQL clusterization can also be an option if the app grows in the future.

#### Asynchronous jobs

We'll use Sidekiq to perform asynchronous jobs. Each job will have it's own worker class. Queues will be stored in Redis.

#### Error Handling

We'll use standard HTTP status code together with our internal code and error messages.

The `Errors::ErrorHandler` should encapsulate the error logic and this module should just be included in specific parts of the API.
 
<img src="https://github.com/chat-app-architecture/architecture_proposal/blob/main/images/error-handling.png" width=500 />

The error message response is as follows:

```ruby
{
    “code”: code,
    “title”: title,
    “status”: status,
}
```

For this project, we’ll raise exceptions from the Services and rescue them in the Controllers. The Controllers essentially only route the requests to the Services and rescue expectations to return customized error messages.

#### Controllers

| Controller                        | Action          | Single responsibility                |
|-----------------------------------|-----------------|--------------------------------------|
| RegistrationsController           | create          | Creates user                         |
| SessionsController                | create          | Authenticates user and returns token |
| SessionsController                | destroy         | Signs out user                       |
| GroupMessagesController           | create          | Creates and broadcasts message       |
| GroupsController                  | index           | Returns a list of groups             |
| GroupsController                  | show            | Returns a specific group             |
| GroupsController                  | create          | Create group                         |


#### Services

For the API, I’ve chosen the Rails Services pattern. They offer the benefit of concentrating the core logic of the application in a separate object, instead of scattering it around controllers. It’s also a good pattern for testing purposes where you can isolate a feature or a set of steps without needing to hit the controller.

| Controller                        | Description                          |
|-----------------------------------|--------------------------------------|
| Groups::CreateService             | Creates group                        |
| Groups::ListService               | Retrieve a list of groups            |
| Groups::ShowService               | Retrieve information about a group   |
| GroupMessages::SendMessageService | Broadcasts message and save in DB    |

Choosing design patterns for your APIs is important.

#### Documentation of API

At the heart of the Swagger tools is the OpenAPI Specification.

I’ve attached a Swagger docs file in the solution. You can copy and paste the code in the Online Swagger Editor: https://editor.swagger.io/.

<img src="https://github.com/chat-app-architecture/architecture_proposal/blob/main/images/swagger-docs.png" width=500 />

#### Rollbar

Rollbar is the option regarding error tracking and monitoring. It offers a good query search & filter tool and lots of useful information on error debugging and replay.

#### Circle CI

Automated testing and deployment will run with the help of Circle CI.

#### Ruby Scanners

Recommendation for any application that you build in Ruby:

- `Brakeman` to check for vulnerable versions of gems in Gemfile.lock.
- `Bundler-audit` for a static analysis tool which checks Ruby on Rails applications for security vulnerabilities.
- `Rubocop`: a static code analyzer and formatter, based on the community Ruby style guide.

#### Tests and Delivery Automation

We'll use the following tools to guarantee the quality of the application:

- TDD using RSpec, FactoryBot and ShouldaMatchers.
- Mock and Stubs to test API requests. We can also use VCR cassettes when needed.
- Continuous Integration and Continuous Delivery using CircleCI.
- QA using Insomnia.

<img src="https://github.com/chat-app-architecture/architecture_proposal/blob/main/images/workflow.png" width=500 />

#### Docker

Docker should be our default tool in any project that we build. `Dockerfile`, `docker-compose.yml` and a `Makefile` are required to work with containers locally and be able to deploy applications via Docker. Some suggestion for the projects’ Makefile:

| Command                   | Description                            |
|---------------------------|----------------------------------------|
| make start                | Start the application                  |
| make specs                | Run all the specs                      |
| make bash                 | access the bash inside the container   |
| make console              | access rails console                   |
| make routes               | access rails routes                    |
