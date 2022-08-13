---
layout: post
resources: context-dependent-imports-using-cls-and-getters
title: Context-Dependent Imports Using CLS and Getters
date: 2020-06-15
footer_content: This article was [originally written for the Trabe medium publication](https://medium.com/trabe/continuation-local-storage-for-easy-context-passing-in-node-js-2461c2120284), a collection of excellent articles written by [the awesome people from trabe.io](https://trabe.io/).
---

As I was reading an article about [exporting getters in CommonJS](https://medium.com/trabe/exporting-getters-in-commonjs-modules-dd8f98b7d85e) from my colleague [David Barral](https://medium.com/@david.barral) the other day, I realized we could use this approach to export dynamically from a CLS context.

Since I recently wrote [a post about AsyncLocalStorage](https://rcoedo.com/blog/2020/05/25/async-local-storage-for-easy-context-passing-in-node-js), I thought that I could probably dig a bit deeper on this subject.

## Refactoring our CLS example

In my previous post, I used this example:

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

We can make it more readable by encapsulating the CLS usage. We move some of the logic to a `context.js` module:

```js
const { AsyncLocalStorage } = require('async_hooks');
const { v4: uuid } = require('uuid');

const defaultStore = new Map();
defaultStore.set('requestId', 'not-in-a-request');

const context = new AsyncLocalStorage();
context.enterWith(defaultStore);

const middleware = (req, res, next) => {
  context.run(new Map(), () => {
    context.getStore().set('requestId', uuid());
    next();
  });
};

module.exports = {
  context,
  middleware,
};
```

We added a couple of lines of code. We defined a default store and assigned it to AsyncLocalStorage with the `enterWith` method. This store will be returned when we are out of the request scope.

This is our `main.js,` now using the new module:

```js
const express = require('express');
const ctx = require('./context.js');

const app = express();
app.use(ctx.middleware);

app.get('/', (req, res) => {
  const id = ctx.context.getStore().get('requestId');
  console.log(`[${id}] request received`);
  res.send('It works!');
});

const port = process.env.PORT || 3000;
app.listen(port, () => console.log(`Express server listening on port ${port}`));
```

## Using the getter magic

The cool thing about getter exports being dynamic is we can encapsulate the logic of taking the `requestId` out of our context in the `context.js` module.

We add the `requestId` getter to the `module.exports,` and the result looks like this:

```js
module.exports = {
  get requestId() {
    return context.getStore().get('requestId');
  },
  middleware,
};
```

Now we can access `requestId` just like any other attribute from that module:

```js
const express = require('express');
const ctx = require('./context.js');

const app = express();
app.use(ctx.middleware);

app.get('/', (req, res) => {
  console.log(`[${ctx.requestId}] request received`);
  res.send('It works!');
});

const port = process.env.PORT || 3000;
app.listen(port, () => console.log(`Express server listening on port ${port}`));
```

If we access `ctx.requestId` out of scope, we get a `not-in-a-request` identifier; otherwise, we get the proper request identifier. We can expand the `context.js` API as much as we want by exporting more getters.

## Careful; this is magic

Exporting getters is an excellent way to make clean APIs, but if we are not aware that we are using them, bad things can happen. Let's check this other scenario:

If we make an HTTP request to the service now, we get this:

```
[not-in-a-request] request received
```

## What's happening here?

We are destructuring the module outside of scope. Since `requestId` is a getter and we are using the destructuring assignment at the top level, we are assigning its value outside the request scope. This causes `not-in-a-request` to be assigned to `requestId`.

## Should you do this?

This is the same question David asked at the end of his post. Honestly, I wasn't a big fan of exporting getters the first time I read about it. For this specific use case, though, I think it makes things more readable and clear.

You can find the example code [in this repository](https://github.com/rcoedo/express-context-dependent-exports).

Enjoy!
