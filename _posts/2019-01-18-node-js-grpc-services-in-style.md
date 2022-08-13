---
layout: post
title: Node.js gRPC Services in Style
date: 2019-01-18
image: node-js-grpc-services-in-style.jpg
footer_content: This article was [originally written for the Trabe medium publication](https://medium.com/trabe/node-js-grpc-services-in-style-222389be991a), a collection of excellent articles written by [the awesome people from trabe.io](https://trabe.io/).
---

In the last few years, gRPC has been gaining much traction. In this post, we will cover the basics of gRPC and jump right into building a Greeter service with [grpc-node](https://github.com/grpc/grpc-node). After that, we're going to rewrite it using [Mali](https://mali.js.org/).

This post will assume no previous gRPC experience, so if you haven't had the time to check what gRPC is about, keep reading!

### What is gRPC?

gRPC is a new take on the RPC paradigm to build APIs that recalls the old SOAP protocol. gRPC and SOAP are similar in that both use a contract that both client and service agree upon. However, gRPC handles reasonably well the known [SOAP weaknesses](https://en.wikipedia.org/wiki/SOAP#Technical_critique) while maintaining its strengths.

Instead of XML, gRPC serializes the messages using [Protocol Buffers](https://developers.google.com/protocol-buffers) (AKA protobuf), a Google algorithm to serialize structured data into binary. After being serialized, the data is sent over HTTP/2.

### Services, rpcs, and messages

A _contract_ is defined using a `.proto` file. A protobuf definition file contains the interface for one or more services, the signatures for every remote procedure call in each service, and the structures of the messages that will be sent to the service (or received from it).

You can find a complete language guide with detailed type definitions in [the Google developers guide](https://developers.google.com/protocol-buffers/docs/proto3).

## Coding a gRPC service with grpc-node

We will build a greeter service from scratch. This is the `greeter.proto` file that we are going to use:

```protobuf
syntax= "proto3";

service HelloWorldService {
  rpc GreetMe (GreetRequest) returns (GreetReply) {}
}

message GreetRequest {
  string name = 1;
}

message GreetReply {
  string reply = 1;
}
```

It should be pretty straightforward to read. Just a quick note; when we define the message fields, we assign a number to every field. These field numbers identify the fields when the message is serialized into binary.

### Loading the proto: static vs. dynamic

The next step is to load our service definition into our server. There are two ways to load a proto definition: statically and dynamically.

Using static generation, we can pre-compile the proto definition into EcmaScript code (or any other supported language, actually) using [the protoc command line tool](https://www.systutorials.com/docs/linux/man/1-protoc/) to get some generated data structures with input validation for the messages.

Dynamic loading the `.proto` file is easier to understand since we can avoid the hassle of compiling the file, but we lose the input validation.

For simplicity's sake, we will use dynamic loading in our example. We are going to use the `@grpc/proto-loader` library to load the proto definition file dynamically:

```js
const protoLoader = require('@grpc/proto-loader');

const proto = protoLoader.loadSync(path.join(__dirname, '..', 'greeter.proto'));
const definition = grpc.loadPackageDefinition(proto);
```

More than one service can be defined in a single proto file. The `grpc.loadPackageDefinition` function loads every service definition defined in the proto file. Using these definitions, we can then attach the implementations to the server:

```js
const grpc = require('@grpc/grpc-js');

const greetMe = (call, callback) => {
  callback(null, { reply: `Hey ${call.request.name}!` });
};

const server = new grpc.Server();

server.addService(definition.GreeterService.service, { greetMe });
```

Since we only have one service with one RPC, that is all we need to do. We are ready to start the server:

```js
server.bindAsync(
  '0.0.0.0:50051',
  grpc.ServerCredentials.createInsecure(),
  (port) => {
    server.start();
  }
);
```

The complete implementation looks like this:

```js
const path = require('path');
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');

const proto = protoLoader.loadSync(path.join(__dirname, '..', 'greeter.proto'));
const definition = grpc.loadPackageDefinition(proto);

const greetMe = (call, callback) => {
  callback(null, { reply: `Hey ${call.request.name}!` });
};

const server = new grpc.Server();

server.addService(definition.GreeterService.service, { greetMe });

server.bindAsync(
  '0.0.0.0:50051',
  grpc.ServerCredentials.createInsecure(),
  (port) => {
    server.start();
  }
);
```

To test our service, we can use [grpcurl](https://github.com/fullstorydev/grpcurl), a command line tool to make requests to gRPC services:

```
> grpcurl -plaintext -proto greeter.proto \
    -d '{ "name": "Jude" }'               \
    0.0.0.0:50051                         \
    GreeterService/GreetMe

# response:
# {
#   "reply": "Hello Jude!"
# }
```

If you prefer a desktop tool, [BloomRPC](https://github.com/bloomrpc/bloomrpc) is also available.

## Using Mali

Mali is a minimalistic Node.js framework for building gRPC services. It looks a lot like the popular HTTP framework Koa.

In Mali, we define remote procedure handlers as a stack of middleware. A request cascades through the middleware, mutating a context object, attaching or parsing data, and triggering side effects.

A Mali middleware is an asynchronous function that has two parameters; the context and the `next` function:

```js
async (ctx, next) => {
  // Do some stuff, maybe.
  // cascade through the rest of the middleware chain
  await next();
  // Do more stuff, or not. ¯\_(ツ)_/¯
};
```

The `next` function is an async function that invokes the next middleware in the middleware chain. Using `next,` you can hook your code around the request.

Let's implement a simple time middleware to illustrate this with an explicit example:

```js
async (ctx, next) => {
  const start = new Date();

  await next();

  const ms = new Date() - start;
  console.log(`[${start.toLocaleString()}] Request ${ctx.name} took ${ms}ms`);
};
```

In this middleware, we take the start time at the beginning of the request, let the rest of the chain process the request, calculate the duration in milliseconds, and print it out.

### Refactor our service into a Mali service

To refactor our previous gRPC server to use Mali instead of grpc-node, we need to convert our `greetMe` handler into a middleware:

```js
// Before
const greetMe = (call, callback) => {
  callback(null, { reply: `Hey ${call.request.name}!` });
};

// After
const greetMe = async (ctx, next) => {
  ctx.res = { reply: `Hey ${ctx.req.name}` };
  await next();
};
```

We then get a Mali instance, passing it the proto file path, and attach the middleware:

```js
const path = require('path');
const Mali = require('mali');

const greetMe = async (ctx, next) => {
  ctx.res = { reply: `Hey ${ctx.req.name}!` };
  await next();
};

const app = new Mali(path.join(__dirname, '..', 'greeter.proto'));

app.use({ greetMe });

app.start('0.0.0.0:50051');
```

As we did before, we can use `grpcurl` to test it.

### Plugging in some middleware

The cool thing about Mali is that it is easy to extend by just plugging new middleware into it. Let's add our previous logging middleware to the service:

```js
const path = require('path');
const Mali = require('mali');

const greetMe = async (ctx, next) => {
  ctx.res = { reply: `Hey ${ctx.req.name}!` };
  await next();
};

const logRequest = async (ctx, next) => {
  const start = new Date();

  await next();

  const ms = new Date() - start;
  console.log(`[${start.toLocaleString()}] Request ${ctx.name} took ${ms}ms`);
};

const app = new Mali(path.join(__dirname, '..', 'greeter.proto'));

app.use(logRequest);
app.use({ greetMe });

app.start('0.0.0.0:50051');
```

That's it! Now, if we make a `GreetMe` call, we will get a nice log message.

## Wrapping it up

We've skimmed through the basics of gRPC, and we've built the most straightforward possible service using grpc-node and Mali.

The Mali version is much more extensible, and the middleware concept is a powerful abstraction that fits really well here.

We've only covered unary operations and haven't even mentioned how to generate the client's code using the proto file, but that is left for future posts.

You can find all [the js code we wrote in this repository](https://github.com/rcoedo/grpc-services-playground).

Enjoy!
