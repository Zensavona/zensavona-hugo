+++
date = "2017-02-20T12:09:14+11:00"
tags = ["react", "jboss", "hot module reloading"]
title = "Using React Hot Module Reloading with JBoss (or any other server software)"
+++

Recently at work I have been developing a JavaScript application which talks to a JBoss (Java) based API. This application is deployed and served by JBoss and previously the way I was doing development was having a `webpack` watch script with `npm`, and then `rsync` the built files to the directory the Java server serves them from. This is mostly because of CORS and JBoss SSO only working for specific domains (and by extension, ports).

Recently I realised that I can actually improve my workflow significantly with HMR (Hot Module Reloading) and that it is totally possible to do without using node to serve your `index.html` (or anything).

### Ok, so how do we do it?

First up you'll need different `index.html` files to host your JS code for development and production. This is true for any HMR setup.

Let's take a look at the [relevant section of the Webpack docs](http://webpack.github.io/docs/webpack-dev-server.html#combining-with-an-existing-server).

What this is basically telling us, is that normally `webpack-dev-server` serves both the inital `index.html` file with a link to a url which serves the hot loaded code, and the actual code as we change it. These are by default served on ports 8080 and 9090. What we want to do is half of that. We still want to serve the hot loaded code, but not the `index.html` as we will serve that ourselves with our own webserver (in my case JBoss).

This guide assumes you have a basic `webpack 1.x` setup already going. If you don't, you can refer to [this guide](http://blog.tamizhvendan.in/blog/2015/11/23/a-beginner-guide-to-setup-react-dot-js-environment-using-babel-6-and-webpack/).

### Installing dependencies

We need to install (if you don't already have them) `html-webpack-plugin`, `webpack-dev-middleware`, `webpack-dev-server` and `webpack-hot-middleware`, `url-loader`, `style-loader`, `file-loader`.

You can just run `npm install --save-dev webpack-hot-middleware webpack-dev-server webpack-dev-middleware html-webpack-plugin style-loader file-loader url-loader`

### Webpack Config

Next we'll update our `webpack.conf.js` to run the dev server and do HMR!

Below is a simple example of a `webpack.conf.js`, you can copy and paste bits that look different into your own.

**This configuration makes a few assumptions:**

- Your built files will be output to `dist/`
- Your source files live in `src/`
- Your application entrypoint is `src/main.{js, jsx}` (can have either js or jsx extension)
- There exists a file in the same folder as this config file called `index.ejs` (we'll get to this in just a sec)

```
var webpack = require('webpack');
var path = require('path');
var HtmlWebpackPlugin = require('html-webpack-plugin');

var BUILD_DIR = path.resolve(__dirname, 'dist');
var APP_DIR = path.resolve(__dirname, 'src');

var config = {
  devtool: 'eval',
  entry: [
    "webpack-dev-server/client?http://localhost:9090",
    "webpack/hot/only-dev-server",
    'babel-polyfill',
    APP_DIR + '/main'
  ],
  output: {
    path: BUILD_DIR,
    filename: 'bundle.js',
    publicPath: "http://localhost:9090/"
  },
  plugins: [
    new webpack.HotModuleReplacementPlugin(),
    new webpack.NoErrorsPlugin(),
    new HtmlWebpackPlugin({
      template: 'index.ejs'
    })
  ],
  devServer: { inline: true },
  module : {
    loaders : [
      {
        test : /\.jsx?$/,
        include : APP_DIR,
        loaders : ['react-hot', 'babel']
      },
      {
        test: /\.css$/,
        loader: 'style!css?sourceMap'
      }, {
        test: /\.woff(\?v=\d+\.\d+\.\d+)?$/,
        loader: "url?limit=10000&mimetype=application/font-woff"
      }, {
        test: /\.woff2(\?v=\d+\.\d+\.\d+)?$/,
        loader: "url?limit=10000&mimetype=application/font-woff"
      }, {
        test: /\.ttf(\?v=\d+\.\d+\.\d+)?$/,
        loader: "url?limit=10000&mimetype=application/octet-stream"
      }, {
        test: /\.eot(\?v=\d+\.\d+\.\d+)?$/,
        loader: "file"
      }, {
        test: /\.svg(\?v=\d+\.\d+\.\d+)?$/,
        loader: "url?limit=10000&mimetype=image/svg+xml"
      }
    ]
  },
  resolve: {
    extensions: ['', '.js', '.jsx', '.css']
  }
};

module.exports = config;
```

### index.ejs

This is basically a template `webpack` uses to inject the markup required for hot loading. It is basically just a html file (it can [output some variables](https://github.com/jantimon/html-webpack-plugin#configuration)) which serves your bundle.

```
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>My Application</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <!-- imported CSS are concatenated and added automatically -->
  </head>
  <body>
    <div id="app"></div>
  </body>
</html>
```

### The HMR Server

Next, we need to install express (`npm install --save-dev express`) and create a file called `server.js`

What this does is actually serve the bundled file for `webpack-dev-server` to send through to the page every time we make a change. I also have mine serve a `style.css` file.

```
var express = require('express');
var app = express();

// Serve application file
app.get('/bundle.js', function(req, res) {
  res.sendFile(__dirname + '/dist/bundle.js');
});

// Serve aggregate stylesheet
app.get('/style.css', function(req, res) {
  res.sendFile(__dirname + '/build/style.css');
});

app.use(express.static('dev-public'));


// Serve index page
app.get('*', function(req, res) {
  res.sendFile(__dirname + '/dist/index.html');
});

if (!process.env.PRODUCTION) {
  var webpack = require('webpack');
  var WebpackDevServer = require('webpack-dev-server');
  var config = require('./webpack.config');

  new WebpackDevServer(webpack(config), {
    publicPath: config.output.publicPath,
    hot: true,
    noInfo: true,
    historyApiFallback: true
  }).listen(9090, 'localhost', function (err, result) {
    if (err) {
      console.log(err);
    }
  });
}
```

### NPM Script & Starting Things Up

Let's make a new entry in the `scripts` section of our `package.json` that looks like this:

```
"server": "webpack --progress -p && node server.js"
```

Then run `npm run server`.

Lastly, we need to copy the `index.html` file from our `dist/` directory to wherever the webserver (in my case, JBoss) is looking for it. This could be in a `.war/` folder/file somewhere on your filesystem. This method works for pretty much any kind of webserver!

**Let me know in the comments below if this worked for you, or if you need any more help with it!**
