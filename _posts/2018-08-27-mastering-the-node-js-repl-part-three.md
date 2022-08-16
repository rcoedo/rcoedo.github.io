---
layout: post
resources: mastering-the-node-js-repl-part-three
title: Mastering the Node.js REPL, Part Three
date: 2018-08-27
footer_content: This article was [originally written for the Trabe medium publication](https://medium.com/trabe), a collection of excellent articles written by [the awesome people from trabe.io](https://trabe.io/).
---

In the second part of this series, we covered the Node.js REPL programmatic API. This will be the last part of the series, and we will write an example express application that takes advantage of that programmatic API.

[All the code for this article lives in this repository](https://github.com/rcoedo/mastering-the-node-js-repl/tree/master/part-3). I strongly recommend you check it out and follow me in this article.

## The express application

We will use a naive express application with a service layer and a controller layer. We won't go into much detail about the express application, but the project structure looks like this:

![example 1](/resources/images/mastering-the-node-js-repl-part-three/example1.png)

Within the project, you can find a `data.js` file that uses [Faker](https://fakerjs.dev/) to generate some mock data. The data will be 20 users and 100 posts associated with these users. The users will have an `id,` `name,` and `email.` The posts will have an `id,` `title,` `body,` and `userId.`

For simplicity, the service layer will only have a bunch of `read` functions, and the controllers will use those services to return the data.
The main script mounts the router and starts the express application:

```js
const express = require('express');
const controllers = require('./controllers');

const app = express();

app.get('/users', controllers.getUsers);
app.get('/users/:userId', controllers.getUser);
app.get('/users/:userId/posts', controllers.getUserPosts);

app.get('/posts', controllers.getPosts);
app.get('/posts/:postId', controllers.getPost);

app.listen(3000, () => console.log('express server listening on port 3000'));
```

In this link, you can check [the complete code for the express application](https://github.com/rcoedo/mastering-the-node-js-repl/tree/master/part-3/express-app).

## Adding a `repl` to the project

Now that we have created our basic express application, we can add a REPL to the project. We make a `dev/repl/` directory and throw in there our code.

This is the code for `dev/repl/repl.js`:

```js
const Repl = require('repl');
const {
  extendWith,
  colorize,
  defineCommands,
  clearRequireCache,
} = require('./utils');
const { sayWelcome, sayBye, sayDoc, prompt } = require('./cli');

// Define a context initializer
const initializeContext = (context) => {
  clearRequireCache();

  extendWith({
    R: require('ramda'),
    services: require('../../services'),
  })(context);
};

sayWelcome();

const repl = Repl.start({ prompt });

defineCommands({
  doc: {
    help: 'Get information about the loaded modules',
    action() {
      this.clearBufferedCommand();
      sayDoc();
      this.displayPrompt();
    },
  },
})(repl);

initializeContext(repl.context);

repl.on('reset', initializeContext);
repl.on('exit', sayBye);
```

If you remember part 2 of the series, this code should sound familiar. The steps we follow are:

1. Define our context initializer function.
2. Print a welcome message.
3. Start the REPL with our custom prompt.
4. Define our commands.
5. Bind our event handlers.

The context initializer clears the `require` cache and re-requires some modules when the `clear` command is invoked. This feature allows us to reload our modules if we make changes to them without restarting the REPL.

We can also bind any library to the context for testing purposes, just like we did with `ramda.`

Lastly, we add a new script to the `package.json` file to run the repl:

```json
"scripts": {
  "start": "nodemon main.js",
  "repl": "node dev/repl/repl.js"
}
```

`yarn repl` will now start the REPL session and load our context:

![example 2](/resources/images/mastering-the-node-js-repl-part-three/example2.png)

## Connecting to a node process using sockets

We are now going to go one step further. Using the [net node library](https://nodejs.org/api/net.html), we will build a REPL server and start it alongside our express server. This will allow us to connect to the server's running process and do anything we want in there.

### The server code

We moved our `dev/repl/` code to `repl/.` The code for the net server is straightforward:

```js
const net = require('net');
const repl = require('./repl');

const server = net.createServer((socket) => {
  start(socket);
});

module.exports = server;
```

We create a new server and invoke the `repl` function passing the socket. The `repl` function starts a new REPL and handles the socket events:

```js
  console.log("repl client connected");
  sayWelcome(socket);

  const repl = Repl.start({
    prompt,
    input: socket,
    output: socket,
    terminal: true,
  });

  // define commands...
  // initialize context...
  // listen repl events...

  socket.on("error", e => {
    console.log(`repl socket error: ${e}`);
  });

  repl.on("exit", () => {
    console.log("repl client disconnected");
    socket.end();
  });
};
```

`input` and `output` are the REPL's streams to read from and write to. `input` is a `stream.Readable` and `output` is a `stream.Writable`. Sockets are duplex streams, so we can use them for `input` and `output.`

We have to handle a couple of extra cases: When the REPL exits, we close the socket, and if there is an error in the socket, we just log it.

### Writing our client's code

Now that we have the server, we need to write the client. We will write a node script that receives a `<HOST:PORT>` as its first argument and connects to the running server using the `net` library.

```js
#!/usr/bin/env node
const net = require('net');

const args = process.argv.slice(2);
if (args.length < 1) {
  console.log('USAGE: repl <HOST:PORT>');
  process.exit(1);
}

const url = args[0];
const [host, port] = url.split(':');

const socket = net.connect(parseInt(port), host);

process.stdin.pipe(socket);
socket.pipe(process.stdout);

socket.on('connect', () => {
  process.stdin.setRawMode(true);
});

socket.on('close', () => {
  process.exit(0);
});

process.on('exit', () => {
  socket.end();
});
```

We need the socket to read from `process.stdin` and to write to `process.stdout`. We can do that by piping `process.stdin` stream to the socket and the socket to `process.stdout`.

On `connect,` we set `raw` mode to `true.` This will force special terminal characters and combinations like `CTRL+C` to not be processed, so they can be sent through the socket and handled by the server.

To conclude, we close the socket when the process exits and exit the process when the socket closes.

We change our `package.json` to include a script that runs the client, and we are ready to go:

```json
"scripts": {
  "start": "nodemon main.js",
  "repl": "./bin/repl-client localhost:3001"
}
```

We start the server:

![example 3](/resources/images/mastering-the-node-js-repl-part-three/example3.png)

We connect to it using the client:

![example 4](/resources/images/mastering-the-node-js-repl-part-three/example4.png)

And that's it! We are running the REPL in the same process that our express server is running in, and we can share the server state or functionality with it.

## Wrapping it up

In this post, we went a bit further and wrote a REPL server for our express application. Congratulations if you made it this far, and I hope these posts were helpful to you.

Enjoy!
