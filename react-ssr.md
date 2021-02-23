---
layout: post
title: 'Server-Side Rendering with React'
category: computers
tags: [javascript, react]
date: 2020-08-11
---

Server rendering in React is often seen as mystical and esoteric, let's shed some light on it!

# Getting Started

Let's take the following typical react client setup:

```javascript file=src/client/index.js
import React from 'react'
import ReactDOM from 'react-dom'
import { BrowserRouter as Router } from 'react-router-dom'
import App from './App'

function Init() {
  return (
    <Router>
      <App />
    </Router>
  )
}

ReactDOM.render(<Init />, document.querySelector('#root'))
```

Note that the setup logic for rendering and routing is separated from the rest of app code.

A trivial example app might look like:

```javascript file=src/client/App.js
import React, { useState } from 'react'
import { Route, useParams } from 'react-router-dom'

export function Hello() {
  return <h3>Hello world!</h3>
}

export function HelloName() {
  const { name } = useParams()
  return <h3>Hello {name}</h3>
}

export default function App() {
  return (
    <>
      <header>appname</header>
      <main>
        <Route exact path="/" children={<Hello />} />
        <Route path="/:name" children={<HelloName />} />
      </main>
    </>
  )
}
```

# Server Rendering

First, let's setup a simple express server that serves static files:

```javascript file=src/index.js
const path = require('path')
const express = require('express')

const { PORT } = process.env
const app = express()

app.use(express.static(path.resolve(__dirname, '../public')))
app.listen(PORT, () => {
  console.log(`Server is running on http://localhost:${PORT}`),
})
```

Combined with the configuration from my [babel-webpack](./babel-webpack) guide (copied below for completeness), you should have a way to generate static files from your client code, and now a way to serve them over http.

```javascript
// webpack.config.js
const path = require('path')
const MiniCssExtractPlugin = require('mini-css-extract-plugin')

const babel = {
  test: /\.jsx?$/,
  exclude: /node_modules/,
  use: 'babel-loader',
}

const css = {
  test: /\.css$/,
  use: [MiniCssExtractPlugin.loader, 'css-loader'],
}

module.exports = {
  entry: { main: './src/client/index' },
  resolve: { extensions: ['.js', '.jsx'] },
  output: {
    path: path.resolve(__dirname, 'public'),
    filename: '[name]-[contentHash:8].js',
  },
  module: {
    rules: [babel, css],
  },
  plugins: [
    new MiniCssExtractPlugin({
      filename: '[name]-[contentHash:8].css',
    }),
  ],
  optimization: {
    splitChunks: {
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]()[\\/]/,
          name: 'vendor',
          chunks: 'all',
        },
      },
    },
  },
}
```

```javascript
// babel.config.js
module.exports = {
  presets: ['@babel/preset-env', '@babel/preset-react'],
}
```

Don't forget to install the build dependencies:

```bash
npm i -D webpack{,-cli} @babel/{core,preset-env,preset-react} {babel,css}-loader mini-css-extract-plugin
```

---

Now, let's add the React rendering logic:

```javascript file=src/index.js highlight=3-6,12-27
import path from 'path'
import express from 'express'
import React from 'react'
import ReactDOMServer from 'react-dom/server'
import { StaticRouter as Router } from 'react-router-dom'
import App from './client/App.jsx'

const { PORT } = process.env
const app = express()

app.use(express.static(path.resolve(__dirname, '../public')))
app.use((req, res) => {
  const routerCtx = {}
  const appHtml = ReactDOMServer.renderToString(
    <Router location={req.url} context={routerCtx}>
      <App />
    </Router>,
  )
  const html = ReactDOMServer.renderToStaticMarkup(
    <html>
      <body>
        <div id="root" dangerouslySetInnerHTML={{ __html: appHtml }} />
      </body>
    </html>,
  )
  res.send(`<!doctype html>${html}`)
})
app.listen(PORT, () => {
  console.log(`Server is running on http://localhost:${PORT}`)
})
```

JSX in our server code won't work out of the box, nor will importing our application code that uses it. Fortunately this is an easy fix using babel during runtime:

```bash
npm i -D @babel/node
```

then try running the server with `PORT=8000 npx babel-node -x .js,.jsx ./src/index.js`

# Again, but with assets

Currently our response isn't including any script or style tags. The best way I know how to solve this, is to ask webpack for [the list of assets it created](https://webpack.js.org/concepts/manifest/), which the [`webpackManifestPlugin`](https://github.com/danethurber/webpack-manifest-plugin) can help us with:

```bash
npm i -D webpack-manifest-plugin
```

```javascript file=webpack.config.js
+const ManifestPlugin = require('webpack-manifest-plugin')

 module.exports = {
   // ...
   plugins: [
+    new ManifestPlugin({
+      writeToFileEmit: true,
+      seed: () => ({
+        "assets": {
+          "scripts": [],
+          "styles": [],
+        },
+      }),
+      generate: (seed, files, entrypoints) => files.reduce((manifest, { path }) => {
+        if (path.endsWith('.js')) manifest.assets.scripts.push(path)
+        else if (path.endsWith('.css')) manifest.assets.styles.push(path)
+        return manifest
+      }, seed()),
+    }),
     // ...
   ],
 }
```

Now we have a file containing the list of assets webpack generated, we can use this to specify our css and js dependencies in our html response:

```javascript file=src/index.js highlight=7,12-17,29-31,34
import path from 'path'
import express from 'express'
import React from 'react'
import ReactDOMServer from 'react-dom/server'
import { StaticRouter as Router } from 'react-router-dom'
import App from './client/App.jsx'
import { assets } from '../public/manifest.json'

const { PORT } = process.env
const app = express()

app.use(express.static(path.resolve(__dirname, '../public')))

app.use((req, res) => {
  const routerCtx = {}
  const appHtml = ReactDOMServer.renderToString(
    <Router location={req.url} context={routerCtx}>
      <App />
    </Router>,
  )
  const html = ReactDOMServer.renderToStaticMarkup(
    <html>
      <head>
        {assets.styles.map(p => (
          <link href={`/${p}`} key={p} rel="stylesheet" type="text/css" />
        ))}
      </head>
      <body>
        <div id="root" dangerouslySetInnerHTML={{ __html: appHtml }} />
        {assets.scripts.map(p => (
          <script src={`/${p}`} key={p} defer type="text/javascript" />
        ))}
      </body>
    </html>,
  )
  res.send(`<!doctype html>${html}`)
})

app.listen(PORT, () => {
  console.log(`Server is running on http://localhost:${PORT}`)
})
```

in order for react to attach event handlers to the existing dom nodes, we have a small change to make in our client setup:

```diff file=src/client/index.jsx
-ReactDOM.render(<Init/>, document.querySelector('#root'))
+ReactDOM.hydrate(<Init/>, document.querySelector('#root'))
```

# The rest of the fiddlings

We can add some integration with react-router and send status codes and redirects at the server level:

```javascript file=src/index.js highlight=8-14
app.use((req, res) => {
  const routerCtx = {}
  const appHtml = ReactDOMServer.renderToString(
    <Router location={req.url} context={routerCtx}>
      <App />
    </Router>,
  )
  if (routerCtx.url) {
    res.redirect(routerCtx.statusCode || 307, routerCtx.url)
    return
  }
  if (routerCtx.statusCode) {
    res.sendStatus(routerCtx.statusCode)
  }
  const html = ReactDOMServer.renderToStaticMarkup(
    <html>
      <head>
        {assets.styles.map(p => (
          <link href={`/${p}`} key={p} rel="stylesheet" type="text/css" />
        ))}
      </head>
      <body>
        <div id="root" dangerouslySetInnerHTML={{ __html: appHtml }} />
        {assets.scripts.map(p => (
          <script src={`/${p}`} key={p} defer type="text/javascript" />
        ))}
      </body>
    </html>,
  )
  res.send(`<!doctype html>${html}`)
})
```

One of the main things we're still missing is meta data in our `<head/>` tags. A few packages exist for this, the best I've found is [react-helmet-async](https://github.com/staylor/react-helmet-async).

```bash
npm i react-helmet-async
```

First in our client setup we have to add the `HelmetProvider`:

```javascript file=src/client/index.js
 import React from 'react'
 import ReactDOM from 'react-dom'
 import { BrowserRouter as Router } from 'react-router-dom'
+import { HelmetProvider } from 'react-helmet-async'

 import App from './App'

 function Init() {
   return (
+    <HelmetProvider>
       <Router>
         <App />
       </Router>
+    </HelmetProvider>
   )
 }
 ReactDOM.hydrate(<Init />, document.querySelector('#root'))
```

and then again in the server:

```javascript file=src/index.js highlight=1,5,7,11,20,24-26
import { HelmetProvider } from 'react-helmet-async'
// ...
app.use((req, res) => {
  const routerCtx = {}
  const helmetCtx = {}
  const appHtml = ReactDOMServer.renderToString(
    <HelmetProvider context={helmetCtx}>
      <Router location={req.url} context={routerCtx}>
        <App />
      </Router>
    </HelmetProvider>,
  )
  if (routerCtx.url) {
    res.redirect(routerCtx.statusCode || 307, routerCtx.url)
    return
  }
  if (routerCtx.statusCode) {
    res.sendStatus(routerCtx.statusCode)
  }
  const { helmet } = helmetCtx
  const html = ReactDOMServer.renderToStaticMarkup(
    <html>
      <head>
        {helmet.title.toComponent()}
        {helmet.meta.toComponent()}
        {helmet.link.toComponent()}
        {assets.styles.map(p => (
          <link href={`/${p}`} key={p} rel="stylesheet" type="text/css" />
        ))}
      </head>
      <body>
        <div id="root" dangerouslySetInnerHTML={{ __html: appHtml }} />
        {assets.scripts.map(p => (
          <script src={`/${p}`} key={p} defer type="text/javascript" />
        ))}
      </body>
    </html>,
  )
  res.send(`<!doctype html>${html}`)
})
```

Now we can update our app to have some common metadata on each page, and use different titles for each route:

```javascript file=src/client/App.js
import React, { useState } from 'react'
import { Route, useParams } from 'react-router-dom'
import { Helmet } from 'react-helmet-async'

export function Hello() {
  return <h3>Hello world!</h3>
}

export function HelloName() {
  const { name } = useParams()
  return (
    <>
      <Helmet>
        <title>Hello {name}!</title>
      </Helmet>
      <h3>Hello {name}!</h3>
    </>
  )
}

export default function App() {
  return (
    <>
      <Helmet defaultTitle="appname" titleTemplate="appname | %s">
        <meta name="example" content="whatever" />
      </Helmet>
      <header>appname</header>
      <main>
        <Route exact path="/" children={<Hello />} />
        <Route path="/:name" children={<HelloName />} />
      </main>
    </>
  )
}
```
