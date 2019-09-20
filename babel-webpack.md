---
layout: post
title: "Getting started with Babel and Webpack"
category: computers
tags: [javascript,nodejs,babel,webpack]
date: 2018-06-06
---

You've heard about them, maybe you don't know why you'd use them, maybe you just want to know how to get started with them. Read on!

# Why?

Imagine you have written some JavaScript code, and you want to depend on some external library, like jQuery or React. You can use separate `<script>` tags in your HTML page but this results in two separate requests, which is slower than one, and becomes unwieldy if/when you need more dependencies.

*Enter Webpack, stage left*

Webpack bundles your dependencies into a single file (it also has options for splitting them into separate "chunks"). It takes a file (it calls it an entry point), looks for your dependencies that use calls to `require`, and recursively scans those files for more dependencies, and gives you a bundle.

Imagine you also want to write your JavaScript using ES2015 (aka ES6) or newer syntax, but want to support older browsers that may not have it.

*Enter Babel, stage right*

Together these tools can help simplify your workflow while giving you a lot of power over how to manage your code.

# How?

Let's create a project, and install Webpack and for the sake of example use jquery as a dependency.

``` bash
mkdir first-bundle
cd first-bundle
npm init -y
npm i -D webpack{,-cli}
npm i -S jquery
```

Create a simple file at `src/index.js`

``` javascript
// src/index.js
const $ = require('jquery')

$('body').prepend('<h1>hello world</h1>')
```

Now use Webpack to compile it, and check the output

``` bash
npx webpack --mode development
less dist/main.js
```

> if `npx webpack` is not available, use `./node_modules/.bin/webpack`

Note the difference when using production mode

``` bash
npx webpack --mode production
less dist/main.js
```

Now let's add babel into this mix

``` bash
npm i -D @babel/core @babel/preset-env babel-loader
```

A small config for babel

``` json
// babel.config.js
module.exports = {
  plugins: [
    '@babel/preset-env',
  ],
}
```

And now we need to make a small webpack config and tell it to use babel 

``` javascript
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: 'babel-loader',
      },
    ],
  },
}
```

Now let's make a new smaller test case just to test Babel

``` javascript
// src/index.js
const test = () => console.log('hello world')
test()
```

And build it one more time and check the output

``` bash
npx webpack --mode production
less dist/main.js
```

Note that Webpack has a small overhead in the bundle size, just under 1KB.

# Level Up

Since the webpack config is a regular javascript file, we're not limited to cramming everything into a single object, we can move things around and extract pieces into variables:

``` javascript
// webpack.config.js
const rules = [
  {
    test: /\.js$/,
    exclude: /node_modules/,
    use: 'babel-loader',
  },
]

module.exports = {
  module: {
    rules,
  },
}
```

Webpack uses a default "entry point" of `src/index.js`, which we can override: 

``` javascript
// webpack.config.js

/* [...] */

const entry = { main: 'src/app.js' }
module.exports = {
  entry,
  module: {
    rules,
  },
}
```

Changing the output path is slightly more complicated:

``` javascript
// webpack.config.js
const path = require('path')

/* [...] */

const entry = { main: 'src/app.js' }
module.exports = {
  entry,
  output: {
    filename: '[name].bundle.js', // file will now be called main.bundle.js
    path: path.resolve(__dirname, 'public'), // changes the output directory to public
  },
  module: {
    rules,
  },
}
```

## React

Install React and the Babel preset:

``` bash
npm i -D @babel/preset-react
npm i -S react react-dom
```

Add the new preset to the babel config:

``` json
// babel.config.js
module.exports = {
  presets: [
    '@babel/preset-env',
    '@babel/preset-react',
  ],
}
```

A small change to the Webpack config:

``` javascript
// webpack.config.js
const rules = [
  {
    test: /\.jsx?$/, // enables babel to work on .jsx extensions
    exclude: /node_modules/,
    use: 'babel-loader',
  },
]

const entry = { main: 'src/app' }

// this tells webpack how to automagically resolve file extensions
const resolve = { extensions: ['.jsx', '.js'] }

module.exports = {
  entry,
  resolve,
  module: {
    rules,
  },
}
```

A small React example:

``` javascript
// src/app.jsx
import React, { useState } from 'react'
import ReactDOM from 'react-dom'

const App = () => {
  const [input, inputChange] = useState('Hello world!')

  return (
    <div>
      <h1>{input}</h1>
      <input value={input} onChange={e => inputChange(e.target.value)} />
    </div>
  )
}

ReactDOM.render(<App />, document.querySelector('body'))
```

Bundle it all up

``` bash
npx webpack --mode production
less public/main.bundle.js
```

---

### TODO
* more recipes?
* webpack-dev-server/middleware
* html-webpack-plugin
* babel-node and compiling for node 
* vendor bundle splitting
* css modules
* postcss
* babel-plugin-lodash
* flow
* typescript
