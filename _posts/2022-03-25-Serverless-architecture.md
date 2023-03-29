---
title: Serverless architecture
date: 2022-03-25 01:00:00 -0500
categories: [笔记]
tags: [CS5289]
pin: true
author: Eraser

toc: true
comments: true
typora-root-url: ../../eraseryao.github.io
math: false
mermaid: true
---

# How the client app connects to api, lambda and DB

## Basic

Lambda software is composed of a number of functions. Creating a serverless app with AWS Lambda is built upon event-based development, so a user’s action serves as a trigger that kickstarts backend and frontend response.

For traditional three-tier client-oriented system, the client make a request to server, then the server side deals with these requests, send some queries to database, and then send response to client. The basic example is online store like following.

![img](/assets/blog_res/2022-03-25-Serverless-architecture.assets/ps.svg)

With this architecture the client can be relatively unintelligent, with much of the logic in the system is implemented by the server application. Such as authentication, page navigation, searching, transactions and exception handling.

### Serverless architecture 

![img](/assets/blog_res/2022-03-25-Serverless-architecture.assets/wkcRENCer2H6otSFiuNfQ-G1fCPzqPISO1Dk0k7lPjvTjgefhb1C0zgAlYjFF2eMFK0-ghG2CepbJf1MH3zCbUV46jduinrzfpxY-lLURkB-jCssCuDp92La2DtZ4yFYDI6il15NGAa67GryE-8OD2E.png)

This is a massively simplified view, but even here we see a number of significant changes:

1. We’ve deleted the authentication logic in the original application and have replaced it with a third-party BaaS service (e.g., Auth0.)
2. Using another example of BaaS, we’ve allowed the client direct access to a subset of our database (for product listings), which itself is fully hosted by a third party (e.g., Google Firebase.) We likely have a different security profile for the client accessing the database in this way than for server resources that access the database.
3. These previous two points imply a very important third: some logic that was in the Pet Store server is now within the client—e.g., keeping track of a user session, understanding the UX structure of the application, reading from a database and translating that into a usable view, etc. The client is well on its way to becoming a [Single Page Application](https://en.wikipedia.org/wiki/Single-page_application).
4. We may want to keep some UX-related functionality in the server, if, for example, it’s compute intensive or requires access to significant amounts of data. In our pet store, an example is “search.” Instead of having an always-running server, as existed in the original architecture, we can instead implement a FaaS function that responds to HTTP requests via an API gateway (described later). Both the client and the server “search” function read from the same database for product data.
5. If we choose to use AWS Lambda as our FaaS platform we can port the search code from the original Pet Store server to the new Pet Store Search function without a complete rewrite, since Lambda supports Java and Javascript—our original implementation languages.

6. Finally, we may replace our “purchase” functionality with another separate FaaS function, choosing to keep it on the server side for security reasons, rather than reimplement it in the client. It too is fronted by an API gateway. Breaking up different logical requirements into separately deployed components is a very common approach when using FaaS.

## API gateways

![img](/assets/blog_res/2022-03-25-Serverless-architecture.assets/ag.svg)

One aspect of Serverless that we brushed upon earlier is an “API gateway.”

An API gateway is an HTTP server where routes and endpoints are defined in configuration, and each route is associated with a resource to handle that route. In a Serverless architecture such handlers are often FaaS functions.

API gateway only exposes single port to client. When an API gateway receives a request, it finds the routing configuration matching the request, and, in the case of a FaaS-backed route, will call the relevant FaaS function with a representation of the original request. The FaaS function will execute its logic and return a result to the API gateway, which in turn will transform this result into an HTTP response that it passes back to the original caller.

Also for the example of pet store, client sends a GET request for searching cats, and API gateway receives this request, find port in router and connect to cats function. Finally cats function will return an HTTP response and sends it back to client.

## RESTfulAPI

### Basic

RESTful API is an interface that two computer systems use to exchange information securely over the internet.

Most business applications have to communicate with other internal and third-party applications to perform various tasks. 

RESTful APIs support this information exchange because they follow secure, reliable, and efficient software communication standards.

Representational State Transfer (REST) is a software architecture that imposes conditions on how an API should work.

### Benefit

RESTful APIs include the following benefits:

#### Scalability

Systems that implement REST APIs can scale efficiently because REST optimizes client-server interactions. Statelessness removes server load because the server does not have to retain past client request information. Well-managed caching partially or completely eliminates some client-server interactions. All these features support scalability without causing communication bottlenecks that reduce performance.

#### Flexibility

RESTful web services support total client-server separation. They simplify and decouple various server components so that each part can evolve independently. Platform or technology changes at the server application do not affect the client application. The ability to layer application functions increases flexibility even further. For example, developers can make changes to the database layer without rewriting the application logic.

#### Independence

REST APIs are independent of the technology used. You can write both client and server applications in various programming languages without affecting the API design. You can also change the underlying technology on either side without affecting the communication.

## Lambda

But In AWS lambda, their architecture are following. AWS Lambda is a compute service that lets you run code without provisioning or managing servers.

Example serverless application architecture using API Gateway

![img](/assets/blog_res/2022-03-25-Serverless-architecture.assets/IL-Sf4gCZP9gWFbJVNphghqjZ87zQWTOAwvC7Y-lwetA4Weoj3CPon-dRTJ1EiZCTYApIGcYup7cLDFvDbr_7HrMbcVt4baMnomS-kcNQqCJjWGCiOiwDG1-3-8e6EU5dZkHNdlgyHU8fRA1DU7lONI.png)

A [Lambda authorizer](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html) is an [AWS Lambda](https://aws.amazon.com/lambda/) function which API Gateway calls for an authorization check when a client makes a request to an API method.

![image-20230329163522443](/assets/blog_res/2022-03-25-Serverless-architecture.assets/image-20230329163522443.png)

In amazon API gateway at AWS, they support WebSockets.

![截屏2023-03-26 19.34.03](/assets/blog_res/2022-03-25-Serverless-architecture.assets/%E6%88%AA%E5%B1%8F2023-03-26%2019.34.03.png)

A WebSocket API is composed of one or more routes. A **route selection expression** is there to determine which route a particular inbound request should use, which will be provided in the inbound request. The expression is evaluated against an inbound request to produce a value that corresponds to one of your route’s *routeKey* values. For example, if our JSON messages contain a property call action, and you want to perform different actions based on this property, your route selection expression might be `${request.body.action}`.

For example: if your JSON message looks like `{“action” : “onMessage” , “message” : “Hello everyone”}`, then the onMessage route will be chosen for this request.

![img](/assets/blog_res/2022-03-25-Serverless-architecture.assets/Gf3LeUPxCRLL7mHEHvNUYC4HZK9lqsn3VPU6KDm-auo72t_xNbSKLT_Vd_BzKrWuPAxXMMizX-Bl0RhDeE7Hb1YXuAkF6tULLgS9nKVstTMIaZ2dVfQtVImDY3gchLCph-gla5d7b_yt5K7UkhOqZYk.png)

Lambda handles:

- Load balancing
- Auto scaling
- Handling failures
- Security isolation
- Operating system management
- Managing utilization

It runs all the logic that behind your website and interfaces with databases, other backend services, or anything else your site needs.

