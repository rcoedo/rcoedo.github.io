---
layout: post
resources: async-local-storage-for-easy-context-passing-in-node-js
title: AsyncLocalStorage for Easy Context Passing in Node.js
date: 2020-05-25
footer_content: This article was [originally written for the Trabe medium publication](https://medium.com/trabe/continuation-local-storage-for-easy-context-passing-in-node-js-2461c2120284), a collection of excellent articles written by [the awesome people from trabe.io](https://trabe.io/).
---

Some months ago, I wrote a post about [storing data in an asynchronous context using continuation local storage](https://rcoedo.com/blog/2019/11/14/continuation-local-storage-for-easy-context-passing-in-node-js) to avoid spreading the request context to every function via parameters. For this purpose, we used `cls-hooked,` a library that implements CLS using the `async_hooks` node API.

With Node.js 13.10, we got a new feature called [AsyncLocalStorage](https://nodejs.org/api/async_hooks.html#async_hooks_class_asynclocalstorage). In this post we're going to learn more about this new feature, and we're going to rebuild our previous solution using **AsyncLocalStorage** this time.

## So, what is AsyncLocalStorage?

AsyncLocalStorage is a new experimental feature in the `async_hooks` package that allows us to store data in asynchronous contexts just like we were doing with cls-hooked. You can think of it as the Node.js alternative to [thread local storage](https://en.wikipedia.org/wiki/Thread-local_storage).

AsyncLocalStorage's API is similar to the `cls-hooked` API, so the examples from our previous post should translate easily.

Let's get into it!

## Building an Express middleware

In our previous post, we started by creating a namespace. Namespaces are used to scope global data:

```js
const { createNamespace } = require('cls-hooked');

const ns = createNamespace('myApp');
```

With the new Node.js API, we create AsyncLocalStorage instances instead:

```js
const { AsyncLocalStorage } = require('async_hooks');

const asyncLocalStorage = new AsyncLocalStorage();
```

Our middleware using `cls-hooked` looked like this:

```js
const requestIdMiddleware = (req, res, next) => {
  ns.run(() => {
    ns.set('requestId', uuid());
    next();
  });
};
```

Using the new Node.js API, it would look like this instead:

```js
const requestIdMiddleware = (req, res, next) => {
  asyncLocalStorage.run(new Map(), () => {
    asyncLocalStorage.getStore().set('requestId', uuid());
    next();
  });
};
```

The `asyncLocalStorage.run` method takes two arguments. The first is our store state, which can be anything we want. In our example, we use a map to have a key/value storage for the request. The second argument is a function. Our state will be retrievable and isolated inside that function, so we call `next()` inside that function to have every other express middleware run within the AsyncLocalStorage context.

Our complete express example would look like this:

```js
const express = require('express');
const { AsyncLocalStorage } = require('async_hooks');
const uuid = require('uuid/v4');

const asyncLocalStorage = new AsyncLocalStorage();

const requestIdMiddleware = (req, res, next) => {
  asyncLocalStorage.run(new Map(), () => {
    asyncLocalStorage.getStore().set('requestId', uuid());
    next();
  });
};

const app = express();

app.use(requestIdMiddleware);

app.get('/', (req, res) => {
  const id = asyncLocalStorage.getStore().get('requestId');
  console.log(`[${id}] request received`);
  res.send('It works!');
});

const port = process.env.PORT || 3000;
app.listen(port, () => console.log(`Express server listening on port ${port}`));
```

## Revisiting CLS performance

In [our previous post](https://rcoedo.com/blog/2019/11/14/continuation-local-storage-for-easy-context-passing-in-node-js), we discussed how the `async_hooks` API adds significant overhead to our application. However, the new AsyncLocalStorage API performs better than the library we were using before.

For context, [here's a link to a benchmark](https://twitter.com/andreypechkurov/status/1234189388436967426) made by [Andrey Pechkurov](https://twitter.com/andreypechkurov).

Enjoy!
