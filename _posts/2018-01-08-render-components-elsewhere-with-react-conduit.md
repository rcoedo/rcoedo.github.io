---
layout: post
title: Render Components Elsewhere with react-conduit
date: 2018-01-08
image: cyberpunk-pipes.jpg
footer_content: This article was [originally written for the Trabe medium publication](https://medium.com/trabe/continuation-local-storage-for-easy-context-passing-in-node-js-2461c2120284), a collection of excellent articles written by [the awesome people from trabe.io](https://trabe.io/).
---

React 16 has brought us [React Portals](https://reactjs.org/docs/portals.html); however, these still require us to deal with DOM elements ourselves. Today we are presenting [react-conduit](https://github.com/trabe/react-conduit), another way of rendering components outside a component's hierarchy.

## TL; DR

`react-conduit` is a library that exports two essential components, `<Inlet />` and `<Outlet />.` Children passed to an inlet will be rendered in every connected outlet.

The different conduits are managed by a `registry,` shared via context by a `<ConduitProvider />.` You can connect two or more React applications with `react-conduit,` creating a single Registry and sharing it among every `<ConduitProvider />.`

## The problem

Some months ago, I had to implement some components for rendering confirmation dialogs and modals. Due to [how CSS stacking contexts work](https://philipwalton.com/articles/what-no-one-told-you-about-z-index/), I went down the rabbit hole.

When you need to render a dialog way down your DOM tree, it is common to find yourself inside a different stacking context from the rest of your UI. In these cases, you need the child to be placed outside their natural container to render it elsewhere.

This problem has been solved by [react-portal](https://github.com/tajo/react-portal), which has been part of the [standard React API since react 16](https://reactjs.org/docs/portals.html). However, the portals API requires us to deal with the management of the DOM nodes ourselves.

```js
render() {
  return ReactDOM.createPortal(
    this.props.children,
    domNode,
  );
}
```

Our UI is pretty complex, and multiple modals, dialogs, and tooltips can be stacked at different levels, so using portals wasn't the best solution. I wanted to be able to define our application's layers in a React way.

## Introducing react-conduit

[react-conduit](https://github.com/trabe/react-conduit) is a library that exports two essential components; `<Inlet />` and `<Outlet />.`

With React portals, you must choose which DOM node you want to render the children in. With react-conduit, you render components inside an inlet, and they will be transported to the connected outlets.

To connect one or more inlets to one or more Outlets, you should label them using the `label` prop. In addition, you can use the props `onConnect` and `onDisconnect` to trigger actions when an inlet or an outlet gets connected or disconnected.

The state of the connected conduits and the connect and disconnect processes are managed by a `registry.` A `registry` is shared by a `<ConduitProvider />` component via context, so the inlets and outlets can connect when they mount or unmount.

You can also connect two or more different React applications by sharing the same registry for each provider, sending elements from one application to the other.

## Practical use cases

### Application layering

Building different layers for an application is one use case where react-conduit works pretty well. If you need separate layers for your tooltips, drop-downs, popovers, or modals, you could do something like this:

```js
import React from 'react';
import { Inlet, Outlet, ConduitProvider } from 'react-conduit';

// We define the outlet, elements will be rendered here
const ModalLayerOutput = () => <Outlet label="modal-layer" />;

// Anything rendered inside a ModalLayer will be sent to its output
const ModalLayer = ({ children }) => (
  <Inlet label="modal-layer">{children}</Inlet>
);

// Our over-simplistic modal component
const Modal = ({ visible, children }) => {
  if (!visible) {
    return null;
  }

  return (
    <ModalLayer>
      <div>{children}</div>
    </ModalLayer>
  );
};

// This is our application's layout.
const Layout = () => (
  <ConduitProvider>
    <div>
      <div>
        <Modal visible>This is the content!</Modal>
      </div>
      <div className="my-layers">
        <ModalLayerOutput />
      </div>
    </div>
  </ConduitProvider>
);
```

You can create as many layers as you want and place them in different parts of your application, solving the previously mentioned CSS stacking problem.

### Detached components

Another great use-case for react-conduit is using it to detach your components, breaking them into different parts to be rendered separately.

Imagine, for example, that you have a component for editing articles. In that component, you render a title, content, and a toolbar, but you want to optionally allow users of your component to render the toolbar anywhere in their application.

You could do something like this:

```js
import React from 'react';
import { Inlet, Outlet } from 'react-conduit';

// The article component accepts a wrapper for the toolbar. By default it's just a div.
const ArticleEditor = ({ article, ToolbarWrapper = div }) => (
  <div>
    <ToolbarWrapper>
      <MyAmazingToolbar />
    </ToolbarWrapper>

    <div className="title">{article.title}</div>
    <div className="content">{article.content}</div>
  </div>
);

// Now if the parent wants to break the article editor in parts,
// it just needs to pass an Inlet as the wrapper and render an outlet
// where needed.
const ParentComponent = () => {
  const article = { id: 1, title: 'Just a title', content: 'Just the content' };
  const ToolbarWrapper = ({ children }) => (
    <Inlet label={`article-editor-${article.id}`}>{children}</Inlet>
  );

  return (
    <div>
      <ArticleEditor article={article} ToolbarWrapper={Wrapper} />

      <div className="my-toolbar-container">
        <Outlet label={`article-editor-${article.id}`} />
      </div>
    </div>
  );
};
```

This is great because the Editor component is not tied to react-conduit; it just accepts a container. This could even be a stateful component, and we would be able to render things in different parts of our application.

### Cross application renders

You may be developing a non-React application that uses React for just some specific parts of it. If this is the case and you need the different react applications to communicate somehow, you could do something like this:

```js
import React from 'react';
import ReactDOM from 'react-dom';
import { Inlet, Outlet, ConduitProvider, Registry } from 'react-conduit';

// We create a new registry
const registry = new Registry();

// And a custom provider that uses that registry
const MyCustomConduitProvider = ({ children }) => (
  <ConduitProvider registry={registry}>{children}</ConduitProvider>
);

// We define two different Apps. The first app will render stuff in the second app
const App1 = () => (
  <MyCustomConduitProvider>
    <Inlet label="conduit1">This will be rendered in app2</Inlet>
  </MyCustomConduitProvider>
);

const App2 = () => (
  <MyCustomConduitProvider>
    <Outlet label="conduit1" />
  </MyCustomConduitProvider>
);

// And finally we render each app in the dom
ReactDOM.render(<App1 />, document.getElementById('app1'));
ReactDOM.render(<App2 />, document.getElementById('app2'));
```

## With great power comes great responsibility

Conduits are powerful, but be aware that you will be rendering elements outside of their natural scope. This might create problems with your CSS. If you are not careful, styles won't cascade as you expect.

We've found that `react-conduit` is one handy tool in our belt, but be aware that conduits will increase the complexity of your app if you are not mindful of them. If you overuse this pattern, your code will become harder to follow.
