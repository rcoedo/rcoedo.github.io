---
layout: post
resources: mocking-your-graphql-server-is-easier-with-gql-mock-server
title: Mocking your GraphQL Server is Easier with gql-mock-server
date: 2018-01-22
footer_content: This article was [originally written for the Trabe medium publication](https://medium.com/trabe/continuation-local-storage-for-easy-context-passing-in-node-js-2461c2120284), a collection of excellent articles written by [the awesome people from trabe.io](https://trabe.io/).
---

<div class="dialog"><strong>UPDATE:</strong> Building a mock server with CORS enabled using the Apollo Server library directly is easy enough now, so I've decided to deprecate <code>gql-mock-server</code>.</div>

Not long ago, I was trying to figure out how to make React and GraphQL play nicely together. I wanted to build some React applications to learn the differences between Relay and `react-apollo.` During this process, I bumped into the need to mock a GraphQL server.

What I needed was relatively simple:

- A GraphQL server with a schema and some resolvers
- To be able to modify the GraphQL context for requests that had a specific header
- To have CORS enabled

It took me more time than expected to find the right tools and libraries to build the server, so I extracted it to a small library; [gql-mock-server](https://github.com/rcoedo/gql-mock-server).

It looks like this:

```js
import gql from 'gql-mock-server';

gql({
  types: `
    type User {
      id: ID
      name: String
    }
    type Query {
      users: [User]
    }
  `,
  resolvers: {
    Query: {
      users: () => [
        { id: '1', name: 'Peter' },
        { id: '2', name: 'Frank' },
      ],
    },
  },
});
```

The server's default GraphQL endpoint is `http://localhost:3002/graphql,` and it has CORS enabled. The `gql` function also accepts an options object as the second argument:

```js
import gql from 'gql-mock-server';

// const types = ...
// const resolvers = ...

gql(
  { types, resolvers },
  {
    context: (req) => ({
      user:
        req.headers.authorization === 'Bearer VALID_TOKEN'
          ? { username: 'john@doe.com' }
          : null,
    }),
    port: 5000,
    endpoint: '/gql',
  }
);
```

You can modify the server port and endpoint using the options object. You can also pass a function that builds the GraphQL context for a given request.

That's all there is to it! I hope you find this tool helpful and that it enables you to start playing with GraphQL in no time.
