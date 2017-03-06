---
layout: post
comments: true
title: "Hot reload with React and Webpack middleware"
description: ""
category:
tags: ["React", "Webpack"]
---
{% include JB/setup %}

<p>Over the week-end I setup to have hot-reload with React, using Express to proxy API calls, and webpack + webpack-dev-middleware for transpilling, processing, and whatever else webpack does. After battling with the various required components, and experiencing emotions ranging from exhilaration to clinical depression, I finally succeeded, and decided to share my journey with others.</p>

<p>Before proceeding further, the end result is a dummy React application. At best, you will be able to change “Hello, World!” to “Hello, World2!” without refreshing your browser…</p>

<p>In my case, I use Express and the fantastic <a href="https://www.npmjs.com/package/request">request</a> module, to first log in our security infrastructure (Tivoli TAM), and pass required cookies with subsequent proxi’ed calls to our backend Restful APIs. Other APIs are behind hybrid (i.e.: half-baked) OAuth2 security services, and I found that using Express was the most flexible way to deal with those pesky situations.</p>

<h4>Let’s get started by briefly describing the main actors involved:</h4>

<ul>
<li><b>Express.</b></li>
<p>I will not cover either the setup of Express in this blog. There are many online resources to get you started with it. Refer to the source code in Github if you are curious.</p>
<li><b>Webpack.</b></li>
<p>ES6 to ES5 transpiling (compiling from source code to source code) is still required, especially in a corporate environment, where proper browser support is critical, even for older versions. Webpack will use Babel, the de-facto standard, which also supports React’s JSX syntax.</p>
<li><b>npm.</b></li>
<p>npm will be our package manager. In the past, I used Bower and Gulp for front-end dependencies management, but webpack is required for react-hot-loader (and it gives more flexibility with less tooling).</p>
<li><b>webpack-dev-middleware.</b></li>
<p>This wrapper for webpack handles compiling your code in memory to serve it over Express in our situation (as opposed to bundling it as files). In watch mode, it delays requests to compile changes, saving the need to wait before refreshing the page. And with hot-reload, you don’t even need to refresh…</p>
<li><b>react-hot-loader.</b></li>
<p>It works by using webpack's Hot Module Replacement (HMR) API, and configuring your application to support hot-reloading. Essentially, we instruct webpack to use React Hot Reload for JSX and JS files.</p>
<li><b>webpack-hot-middleware.</b></li>
<p>This module concerns itself with using said HMR API to connect your browser to webpack-dev-middleware, and propagate updates. React Hot Loader is thus still required to enable hot-reloading in your application, and correctly process these updates.</p>
</ul>

<h4>So let’s dive in, with setting up React with Webpack:</h4>

<pre><code>
# npm install --save react react-dom bootstrap react-bootstrap babel-preset-react
# npm install --save-dev webpack css-loader style-loader file-loader url-loader babel-core babel-loader babel-preset-es2015
</code></pre>

<p>Webpack should aslo be installed globally:</p>

<pre><code>
# npm install -g webpack
</code></pre>

Configure webpack as follow -- <em>webpack.config.js</em>:

<pre><code>
  const path = require('path');
  const webpack = require('webpack');

  module.exports = {
    entry: [
      './src/main',
    ],
    resolve: {
      extensions: ['.js', '.jsx'],
    },
    output: {
      path: path.resolve(__dirname, './app'),
      publicPath: '/app/',
      filename: 'bundle.js',
    },
    module: {
      loaders: [
        {
          test: /.jsx?$/,
          loader: 'babel-loader',
          exclude: /node_modules/,
          query: {
            presets: ['es2015', 'react'],
          },
        },
        {
          test: /\.css$/,
          loader: 'style-loader!css-loader',
        },
        {
          test: /\.png$/,
          loader: 'url-loader?limit=100000',
        },
        {
          test: /\.jpg$/,
          loader: 'file-loader',
        },
        {
          test: /\.(woff|woff2)(\?v=\d+\.\d+\.\d+)?$/,
          loader: 'url?limit=10000&mimetype=application/font-woff',
        },
        {
          test: /\.ttf(\?v=\d+\.\d+\.\d+)?$/,
          loader: 'url?limit=10000&mimetype=application/octet-stream',
        },
        {
          test: /\.eot(\?v=\d+\.\d+\.\d+)?$/,
          loader: 'file',
        },
        {
          test: /\.svg(\?v=\d+\.\d+\.\d+)?$/,
          loader: 'url?limit=10000&mimetype=image/svg+xml',
        },
      ],
    },
  };
</code></pre>

<p>The source of the application is extremely simple:</p>

``` html
import React from 'react';
import ReactDOM from 'react-dom';

const render = Component => {
  ReactDOM.render(
    <h1>Hello, World!</h1>,
    document.getElementById('root'),
);
};

render();
```

<p>Same with the HTML file:</p>

``` html
<!DOCTYPE html>
<html>
<head lang="en">
	<meta charset="UTF-8">
	<title>React Webpack-dev-middleware Hot-reload</title>
</head>
<body>
	<div id="root" class="container"></div>
	<script src="bundle.js"></script>
</body>
</html>
```


<p>Now we can test our setup, to ensure Webpack is properly configured:</p>

<pre><code>
# webpack —progress —color —watch —display-error-details
</code></pre>

<p>You should see no errors, and webpack should generate a <em>app/bundle.js</em> file.</p>

<h4>Setting up webpack with Express:</h4>

<p>Let's install webpack-dev-server and webpack-dev-middleware:</p>

<pre><code>
npm install -g webpack-dev-server
npm install --save-dev express request webpack-dev-middleware body-parser
</code></pre>

<p>Add the following lines to your <em>server.js</em> (refer to the code in Github for the complete file):</p>

<pre><code>
const webpackDevMiddleware = require('webpack-dev-middleware');
const middlewareHotReload = require('webpack-hot-middleware');
const webpack = require('webpack');
const webpackConfig = require('./webpack.config');

// Webpack
const compiler = webpack(webpackConfig);

app.use(webpackDevMiddleware(compiler, {
  publicPath: '/app', // Same as `output.publicPath` in most cases.
}));
</code></pre>

<p>Now starting Express will serve the code without first building <em>bundle.js</em> as a file (the extra arguments should be credentials in my case, but you can comment the login session in the code if you want to try it):</p>

<pre><code>
# node server.js foo bar
</code></pre>

<p>We can now access the page on <em>https://localhost:8223/app</em></p>

<h4>Setting up hot-reload:</h4>

<p>Let's install the required components:</p>

<pre><code>
npm install —save-dev react-hot-loader@next
nom install —save-dev webpack-hot-middleware
</code></pre>

<p>We must now cvonfigure webpack and our application. We modify our application to support hot-reload for React:</p>

``` html
import React from 'react';
import ReactDOM from 'react-dom';
import { AppContainer } from 'react-hot-loader';

const render = Component => {
  ReactDOM.render(
    <AppContainer>
      <h1>Hello, World3!</h1>
    </AppContainer>,
    document.getElementById('root'),
);
};

render();

if (module.hot) {
  module.hot.accept(() => { render(); });
}
```
<p>And finally we modify the webapck configuration:</p>

<pre><code>
const path = require('path');
const webpack = require('webpack');

module.exports = {
  entry: [
    'react-hot-loader/patch',
    'webpack-hot-middleware/client?path=/__webpack_hmr&timeout=20000',
    './src/main',
  ],
  resolve: {
    extensions: ['.js', '.jsx'],
  },
  output: {
    path: path.resolve(__dirname, './app'),
    publicPath: '/app/',
    filename: 'bundle.js',
  },
  plugins: [
    new webpack.HotModuleReplacementPlugin(),
    new webpack.NoEmitOnErrorsPlugin(),
  ],
  module: {
    loaders: [
      {
        test: /.jsx?$/,
        loader: 'babel-loader',
        exclude: /node_modules/,
        query: {
          presets: ['es2015', 'react'],
          plugins: [
            'react-hot-loader/babel',
          ],
        },
      },
      {
        test: /\.css$/,
        loader: 'style-loader!css-loader',
      },
      {
        test: /\.png$/,
        loader: 'url-loader?limit=100000',
      },
      {
        test: /\.jpg$/,
        loader: 'file-loader',
      },
      {
        test: /\.(woff|woff2)(\?v=\d+\.\d+\.\d+)?$/,
        loader: 'url?limit=10000&mimetype=application/font-woff',
      },
      {
        test: /\.ttf(\?v=\d+\.\d+\.\d+)?$/,
        loader: 'url?limit=10000&mimetype=application/octet-stream',
      },
      {
        test: /\.eot(\?v=\d+\.\d+\.\d+)?$/,
        loader: 'file',
      },
      {
        test: /\.svg(\?v=\d+\.\d+\.\d+)?$/,
        loader: 'url?limit=10000&mimetype=image/svg+xml',
      },
    ],
  },
};
</code></pre>

<p>That's it! Now you can change the code in <em>main.jsx</em>, and watch the browser automatically refresh the page for you.</p>
