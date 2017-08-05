---
layout: post
title: 'Introducing Second: a framework for mostly-static React applications'
categories:
- Web Development
- React
tags:
- react
- preact
- static
- server-rendering
author: Joseph Wynn
published: true
extra_head:
  - <link rel="stylesheet" href="/css/highlight.css">
---

## TL;DR

- [Second](https://github.com/wildlyinaccurate/second) is a framework for building React applications where the data is fetched on the server and most of the components do not change after the first render.
- Components can declare their data dependencies in a way similar to [Relay](https://facebook.github.io/relay/).
- Everything is rendered only on the server by default; components can opt-in to client-side rendering using the component dehydrator.
- Second works with any React-like library including [Preact](https://preactjs.com/).

## Why build this framework?

While using React to build the [BBC News front page](/introducing-a-faster-bbc-news-front-page/) and several other mostly-static applications, a common theme emerged: only a small number of components on these pages require client-side JavaScript to function. Bundling the entire page to render on the client does not provide an optimal experience — especially for users on low-powered devices or low-speed connections. Instead, selectively bundling components enables development teams to take advantage of React's event system and stateful components while reducing bundle size and minimizing CPU overhead for client-side rendering.

Second is not a large or complex framework. It is the result of combining several simple and well-tested techniques—

- Server-side rendering
- Container components for declarative data requirements
- Selective client-side rendering

—and combining them into a single package that can be easily reused across multiple applications.<!--more-->

## How does it work?

### Server-side rendering

At its simplest, Second is just a wrapper around server-side rendering:

```js
import VDom from 'react'
import VDomServer from 'react-dom/server'
import second from 'second'
import MyComponent from './components/my-component'

second.init({ VDom, VDomServer })

const props = {
  // ...
}

second.render(MyComponent, props).then((result) => {
  console.log('Rendered HTML:', result)
})
```

### Data-driven components

At the heart of Second is [second-container](https://github.com/wildlyinaccurate/second/tree/master/packages/second-container). Inspired by Facebook's [Relay](https://facebook.github.io/relay/), it is a higher-order component that allows components to declare their data requirements. Unlike Relay which works only with GraphQL, second-container works with regular HTTP requests out of the box and supports custom data fetchers as well.

```js
const ComponentWithData = second.createContainer(MyComponent, {
  data: (props) => ({
    // Each key is provided to the wrapped component as a prop
    response: {
      // Use props to dynamically build API endpoints
      uri: `http://api.duckduckgo.com/?q=${props.query}&format=json`
    }
  })
})
```

Containers can be used at any level of the render tree, allowing leaf nodes to fetch their own data. For static render trees, this typically scales better than fetching data at the root and passing it as a prop down through the entire tree.

## Partial, opt-in browser rendering

[second-dehydrator](https://github.com/wildlyinaccurate/second/tree/master/packages/second-dehydrator) is a higher-order component that serialises ("dehydrates") a component's props and renders them into an inline `<script>` tag. The component can then be hydrated in the browser with those props, and rendered as a regular client-side React component.

Take this "top stories" BBC News component, for example. It is static except for the live-updating promo in the bottom-right corner.

{% responsive_image path: assets/bbc-news-top-stories-live-component.png bleed: true alt: "BBC News top stories component, containing a live-updating promo" %}

This component can be bundled efficiently by dehydrating just the live component:

```jsx
import { createDehydrator } from 'second-dehydrator'
import StaticPromo from './components/static-promo'
import LivePromo from './components/live-promo'

const dehydrate = createDehydrator(React.createElement)
const DehydratedLivePromo = dehydrate(LivePromo)

const TopStories = (props) =>
  <div className="nw-c-top-stories">
    <StaticPromo item={props.items[0]} />
    <StaticPromo item={props.items[1]} />
    <StaticPromo item={props.items[2]} />
    <StaticPromo item={props.items[3]} />
    <StaticPromo item={props.items[4]} />
    <DehydratedLivePromo item={props.items[5]} />
  </div>
```

To hydrate this component in the browser, second-dehydrator takes a component map and a function to call when the component is hydrated. As the hydrator finds dehydrated `LivePromo` components and their corresponding props, it passes them to the function where they can be rendered them with React.

```js
import React from 'react'
import ReactDOM from 'react-dom'
import { hydrate } from 'second-dehydrator'

const renderComponent = (HydratedComponent, props, containerElement) => {
  ReactDOM.render(
    React.createElement(HydratedComponent, props),
    containerElement
  )
}

const componentMap = {
  LivePromo: () => import(/* webpackMode: "eager" */ './components/live-promo')
}

hydrate(renderComponent, componentMap)
```