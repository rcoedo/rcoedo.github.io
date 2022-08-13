---
layout: post
title: Detecting Node.js Active Handles with wtfnode
date: 2019-08-12
image: detecting-node-js-active-handles-with-wtfnode.jpg
footer_content: This article was [originally written for the Trabe medium publication](https://medium.com/trabe/continuation-local-storage-for-easy-context-passing-in-node-js-2461c2120284), a collection of excellent articles written by [the awesome people from trabe.io](https://trabe.io/).
---

Not long ago, I was running some tests for a Node.js project, and for some reason, they ran forever. Something was preventing the process from terminating. It took me some time to realize that it was an interval not being correctly cleared, but I learned a lot in the debugging process.

How can we find out what bits of code are causing our programs to starve forever? Why is this happening?

## The event loop

Node.js uses an event-driven architecture. The core of the event loop is implemented using a C library called [libuv](http://docs.libuv.org/en/v1.x/), which provides asynchronous I/O polling. Node.js also has event queues to enforce things like `process.nextTick()` and implement promises.
If you dig into libuv's code, you can find [this function](https://github.com/libuv/libuv/blob/2480b6158a3a21da564bdb565c4db827df176a4e/src/unix/core.c#L340):

```c
static int uv__loop_alive(const uv_loop_t* loop) {
  return uv__has_active_handles(loop) ||
         uv__has_active_reqs(loop) ||
         loop->closing_handles != NULL;
}
```

This bit is what keeps Node.js running forever. It says, "if there are any active handles, any active requests, or any closing handles, we must keep running." But what are handles and requests?

## Handles and requests

[Handles and requests](http://docs.libuv.org/en/v1.x/design.html#handles-and-requests) are two abstractions provided by libuv. A handle is an object that can do something while it is active. Examples of handles are I/O devices, signals, timers, or processes.

Requests represent short-lived operations, and they can operate over handles. An example could be a `write` request writing into a file.

## Debugging our code

Node.js exposes two undocumented functions to query active handles and requests. The functions are `process._getActiveHandles()` and `process._getActiveRequests()`. They provide detailed information, but that information is rather cryptic.

```js
setTimeout(() => {
  console.log("Look mom, I'm inside a timeout!");
}, 1000);

console.log('-------------------');
console.log('handles:', process._getActiveHandles());
console.log('requests:', process._getActiveRequests());
console.log('-------------------\n');
```

If you are curious, [here is the output of the this snippet](https://gist.github.com/rcoedo/bef1a5e8d4fd430470acdb59aea1b427).

<div class="dialog">Since Node.js@11 timers are not included in the result of getActiveHandles. More info on <a href="https://github.com/nodejs/node/issues/25806">this github issue</a>.</div>

## Wtf, node?

[wtfnode](https://www.npmjs.com/package/wtfnode) is a small package that pretty-prints the information of `getActiveHandles()`. For newer versions of Node.js, it also uses the [async_hooks](https://nodejs.org/api/async_hooks.html) API to track timers.

The usage is relatively simple:

```js
const wtf = require('wtfnode');

console.log('First dump');
console.log('-------------------');
wtf.dump();
console.log('-------------------\n');

setTimeout(() => {
  console.log("Look mom, I'm inside a timeout!");
}, 1000);

console.log('Second dump');
console.log('-------------------');
wtf.dump();
console.log('-------------------\n');
```

The dump function will output something like this:

```sh
$ node index.js
First dump
------------------------
[WTF Node?] open handles:
- File descriptors: (note: stdio always exists)
  - fd 1 (tty) (stdio)
------------------------

Second dump
------------------------
[WTF Node?] open handles:
- File descriptors: (note: stdio always exists)
  - fd 1 (tty) (stdio)
- Timers:
  - (1000 ~ 1000 ms) (anonymous) @ /Users/rcoedo/Workspace/src/github.com/rcoedo/wtfnode/index.js:8
------------------------
```

Node.js always has the `stdio` file descriptor open. In the second dump, we can see the timer, the file, and the line number where the timer was set.

With this information, you should be able to locate the precise line of code that is causing you trouble!
