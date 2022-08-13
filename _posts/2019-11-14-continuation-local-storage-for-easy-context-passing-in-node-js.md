---
layout: post
resources: continuation-local-storage-for-easy-context-passing-in-node-js
title: Continuation Local Storage for easy context passing in Node.js
date: 2019-11-14
footer_content: This article was [originally written for the Trabe medium publication](https://medium.com/trabe/continuation-local-storage-for-easy-context-passing-in-node-js-2461c2120284), a collection of excellent articles written by [the awesome people from trabe.io](https://trabe.io/).
---

Recently I've been working on a Node.js project where we needed to keep track of which requests were generating calls to some other pieces of code. Passing a request identifier around was not viable since it would require changing too many APIs.

We solved the problem using [cls-hooked](https://github.com/jeff-lewis/cls-hooked#readme), a small package that uses the [async_hooks](https://nodejs.org/api/async_hooks.html) experimental Node.js API to implement _Continuation Local Storage_.

## What is Continuation Local Storage?

Some languages have _Thread Local Storage_. This is a way to attach global data to the thread itself. Due to the asynchronous and single-threaded nature of Node.js, this is not useful. Instead, we use _Continuation Local Storage_ (CLS).

CLS is a mechanism that allows the attachment of data to the current asynchronous execution context. It uses `async_hooks` to keep track of asynchronous context changes and to load and unload the data associated with it.

## Building an Express middleware

First, we need to create a namespace. Namespaces are a way to scope global data, so we don't get values from somebody else. We make a namespace with the `createNamespace` function.

```js
const { createNamespace } = require('cls-hooked');

const ns = createNamespace('myApp');
```

Our express middleware will be pretty straightforward. We will generate an `uuid` and store it in the namespace.

```js
const requestIdMiddleware = (req, res, next) => {
  ns.run(() => {
    ns.set('requestId', uuid());
    next();
  });
};
```

As you can see, the `set` method is invoked within a `run` block. The `requestId` value will be available from any code inside that `run` block, which is why we are also calling `next()` inside the `run` block. Calling `next()` inside the `run` block will make the `requestId` available for any other middleware and controller in our express application.

We can now build our complete express application and use the `requestId` anywhere:

```js
const express = require('express');
const { createNamespace } = require('cls-hooked');
const uuid = require('uuid/v4');

const ns = createNamespace('myApp');
const requestIdMiddleware = (req, res, next) => {
  ns.run(() => {
    ns.set('requestId', uuid());
    next();
  });
};

const app = express();
app.use(requestIdMiddleware);

app.get('/', (req, res) => {
  console.log(`[${ns.get('requestId')}] request received`);
  res.send('It works!');
});

const port = process.env.PORT || 3000;
app.listen(port, () => console.log(`Express server listening on port ${port}`));
```

## A word on performance

Even though the solution presented above is clean and requires little coding, it has a significant drawback; The usage of the `async_hooks` API.

The async_hooks API is still experimental, so it could change in the future. What is worse is that enabling it has a considerable impact on performance. This is currently being discussed in [this GitHub issue](https://github.com/nodejs/benchmarking/issues/181).

[Benedikt Meurer](https://medium.com/@bmeurer) wrote [this great benchmark](https://github.com/bmeurer/async-hooks-performance-impact) comparing performances with and without `async_hooks` enabled. If you are writing critical code, you probably want to avoid `async_hooks.` You will also want to stay away from `async_hooks` if you're writing a library since you could impact performance for your users just for your convenience. If the problem you're dealing with does not match either of these scenarios, CLS may be a great way of making things simpler.

Enjoy!
