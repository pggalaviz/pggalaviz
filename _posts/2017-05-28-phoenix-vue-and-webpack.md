---
layout: post
title:  "Configuring a Phoenix app with Vue.js & Webpack"
author: Pedro G. Galaviz
date:   2017-05-23 12:00:00 -0600
comments: true
tags: "elixir, phoenix, vue, webpack"
---

With the new project architecture proposed for Phoenix 1.3 there's a lack of information on how to set a custom frontend.
So, in this post I'll set a Vue.js app for the front, served by webpack.

Why webpack? To me Brunch is not enough, the tooling that supports Vue is currently better on
Webpack so we'll use it.

## Getting started

To get this settings you should have Phoenix version **~1.3** installed. I you don't have it, at the
moment of this writing, you can just run:

{% highlight shell %}
mix archive.install https://github.com/phoenixframework/archives/raw/master/phx_new.ez
{% endhighlight %}

Assuming you have Mix and Elixir installed, right?

Then let's create a new Phoenix project by running (on your desired path):

{% highlight shell %}
mix phx.new phoenix_vue
{% endhighlight %}

And select **no** when asking to install dependencies, this will make easier to change settings. Notice that we're not running `mix phoenix.new` anymore.

We could have added `--no-brunch` to our command but it's easier this way since we can just delete
the brunch configuration and dependencies, plus it will create the **assets** folder (where our Vue
app will live) for us.

Then let's move to our project directory and install Phoenix deps.

{% highlight shell %}
cd phoenix_vue
mix deps.get
{% endhighlight %}

No we can start with our configuration, first we'll visit the **assets** directory and remove the `brunch-config.js` file:

{% highlight shell %}
cd assets
rm brunch-config.js
{% endhighlight %}

And while here, I'll rename the **js** folder to **src** (just a personal preference) and remove the
**css** folder.

{% highlight shell %}
rm -rf css
{% endhighlight %}

Lets update our `package.json` file to look like this (basically remove al brunch dependencies):

{% highlight json %}
{
  "repository": {},
  "license": "MIT",
  "scripts": {
  },
  "dependencies": {
    "phoenix": "file:../deps/phoenix",
    "phoenix_html": "file:../deps/phoenix_html"
  },
  "devDependencies": {
  }
}
{% endhighlight %}

We're now ready to fetch our personalized dependencies (you can use npm or yarn):

{% highlight shell %}
yarn add vue
{% endhighlight %}

And the dev dependencies too:
{% highlight shell %}
yarn add -D babel-core babel-loader babel-preset-stage-2 copy-webpack-plugin css-loader
extract-text-webpack-plugin file-loader image-webpack-loader style-loader url-loader
vue-loader vue-style-loader vue-template-compiler webpack webpack-merge
{% endhighlight %}

We can also add suport for **sass** and **stylus** preprocessors:

{% highlight shell %}
yarn add -D node-sass sass-loader stylus stylus-loader
{% endhighlight %}

Let's add some scripts to our `package.json` file:

{% highlight json %}
{
  // ...
  "scripts": {
    "start": "yarn run dev",
    "dev": "MIX_ENV=dev webpack --watch-stdin --progress --color",
    "deploy": "MIX_ENV=prod webpack -p"
  },
  // ...
}
{% endhighlight %}

And then let's create our main Webpack configuration file on the **assets** folder:

{% highlight shell %}
touch webpack.config.js
{% endhighlight %}

Open it and copy this:

{% highlight javascript %}
'use strict'

// Modules
const path = require('path')
const webpack = require('webpack')
const ExtractTextPlugin = require('extract-text-webpack-plugin')
const CopyWebpackPlugin = require('copy-webpack-plugin')

// Environment
const Env = process.env.MIX_ENV || 'development'
const isProd = (Env === 'production')

function resolve (dir) {
  return path.join(__dirname, dir)
}

module.exports = (env) => {
  const devtool = isProd ? '#source-map' : '#cheap-module-eval-source-map'

  return {
    devtool: devtool,
    entry: {
      app: './src/main.js'
    },
    output: {
      path: path.resolve(__dirname, '../priv/static'),
      filename: 'js/[name].js'
    },
    resolve: {
      extensions: ['.js', '.vue', '.json', '.css', '.scss', '.styl'],
      alias: {
        'vue$': 'vue/dist/vue.esm.js',
        '@': resolve('src')
      }
    },
    module: {
      rules: [
        {
          test: /\.vue$/,
          loader: 'vue-loader',
          options: {
            loaders: {
              css: ExtractTextPlugin.extract({
                use: ['css-loader', 'sass-loader', 'stylus-loader'],
                fallback: 'vue-style-loader'
              })
            }
          }
        }, {
          test: /\.js$/,
          exclude: /node_modules/,
          loader: 'babel-loader',
          include: [resolve('src'), resolve('test')]
        }, {
          test: /\.(gif|png|jpe?g|svg)$/i,
          exclude: /node_modules/,
          loaders: [
            'file-loader?name=images/[name].[ext]',
            {
              loader: 'image-webpack-loader',
              options: {
                query: {
                  mozjpeg: {
                    progressive: true
                  },
                  gifsicle: {
                    interlaced: true
                  },
                  optipng: {
                    optimizationLevel: 7
                  },
                  pngquant: {
                    quality: '65-90',
                    speed: 4
                  }
                }
              }
            }
          ]
        }, {
          test: /\.(ttf|woff2?|eot|svg)$/,
          exclude: /node_modules/,
          query: { name: 'fonts/[hash].[ext]' },
          loader: 'file-loader'
        }
      ]
    },
    plugins: isProd ? [
      new ExtractTextPlugin({
        filename: 'css/[name].css',
        allChunks: true
      }),
      new CopyWebpackPlugin([{
        from: './static',
        to: path.resolve(__dirname, 'priv/static'),
        ignore: ['.*']
      }]),
      new webpack.optimize.UglifyJsPlugin({
        compress: {
          warnings: false
        },
        sourceMap: true,
        beautify: false,
        comments: false
      })
    ] : [
      new ExtractTextPlugin({
        filename: 'css/[name].css',
        allChunks: true
      }),

      new CopyWebpackPlugin([{
        from: './static',
        to: path.resolve(__dirname, 'priv/static'),
        ignore: ['.*']
      }])
    ]

  }
}
{% endhighlight %}

Go to the **src** directory and create the **components** folder and some files:

{% highlight shell %}
cd src
mkdir components
touch App.vue main.js components/Hello.vue
{% endhighlight %}

Edit the 3 files to look like these:

{% highlight javascript %}
// => main.js
import Vue from 'vue'
import App from './App'

Vue.config.productionTip = false

new Vue({
  el: '#app',
  template: '<App/>',
  components: { App }
})
{% endhighlight %}

{% highlight html %}
<!-- App.vue -->
<template>
    <div id="app">
        <header>
            <h1>Phoenix Vue</h1>
        </header>
        <main>
            <hello></hello>
        </main>
    </div>
</template>

<script>
    import Hello from './components/Hello'
    export default {
        name: 'app',
        components: {
            Hello
        }
    }
</script>

<style>
    #app {
          font-family: 'Avenir', Helvetica, Arial, sans-serif;
          -webkit-font-smoothing: antialiased;
          -moz-osx-font-smoothing: grayscale;
          color: #2c3e50;
    }
    main {
        margin-top: 60px;
    }
</style>
{% endhighlight %}
{% highlight html %}
<!-- components/Hello.vue -->
<template>
    <h3>Hello from Vue.js!</h3>
    <p>{% raw %}{{msg}}{% endraw %}</p>
</template>

<script>
    export default {
        name: 'hello',
        data() {
            return {
                msg: 'Simple example using Phoenix, Vue and Webpack.'
            }
        }
    }
</script>

<style scoped>
    p {
        margin-top: 40px;
    }
</style>
{% endhighlight %}

Finally let's add our `.babelrc` file to the **assets** folder:

{% highlight javascript %}
{
  "presets": [
    "stage-2"
  ],
  "comments": false,
  "env": {
    "development": {
      "presets": ["stage-2"]
    }
  }
}
{% endhighlight %}

For now, we're done with the Vue & Webpack part, now lets configure our Phoenix app.

Return to our main project folder `phoenix_vue`. First we'll edit our `.gitignore`, find the 'Static
artifacts' section and add the following:

{% highlight text %}
# Static artifacts
/assets/node_modules
/assets/dist/
/assets/npm-debug.log
/assets/yarn-debug.log
/assets/yarn-error.log
/assets/.sass-cache
/assets/.tern-port
{% endhighlight %}

Next we're going to edit the `config/dev.exs` file:

{% highlight elixir %}
# ...
  watchers: [
    node: ["node_modules/.bin/webpack-dev-server", "--inline", "--colors", "--hot", "--stdin", "--host", "localhost", "--port", "8080", "--public", "localhost:8080",
           cd: Path.expand("../assets", __DIR__)]
  ]
# ...
{% endhighlight %}

That's it for config, now lets update our Phoenix views:

{% highlight shell %}
cd lib/phoenix_vue/web
{% endhighlight %}

Let's edit the `templates/layout/app.html.eex` file and clean the body.

{% highlight html %}
<body>
  <main role="main">
    <%= render @view_module, @view_template, assigns %>
  </main>

  <script src="<%= static_path(@conn, "/js/app.js") %>"></script>
</body>
{% endhighlight %}

And the `templates/page/index.html.eex` file to simply show:

{% highlight html %}
<div id="app"></div>
{% endhighlight %}

And that's it! If we go to the project root path and run `iex -S mix phx.server` we would se our
Vue.js content served by Phoenix.










