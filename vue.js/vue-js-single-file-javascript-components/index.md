# Vue.js Single-File JavaScript Components In The Browser

Browser support for native JavaScript modules is finally happening. The latest versions of Safari and Chrome support them, Firefox and Edge will soon too.

One of the cool things about JavaScript modules for Vue.js users is that they allow you to organize your components into their own files without any kind of build step required. 

In this article, I'm going to show you how to write a single-file component as a JavaScript module and use it in a Vue.js app. You can do this all in the browser without any Babel or Webpack!

> When I say "single-file component" I'm talking about a single JavaScript file which exports a complete component definition. I'm not talking about the single *.vue* file you're used to. Sorry if you're disappointed. But I still think this is pretty cool, so check it out.

> *Note: this article was originally posted [here on the Vue.js Developers blog](https://vuejsdevelopers.com/2017/09/24/vue-js-single-file-javascript-components/?jsdojo_id=cjs_sjc) on 2017/09/24*

## Project setup

Let's use the vue-cli *simple* template to do this. That's right, the one without any Webpack ;)

```
$ vue init simple sfc-simple
```

> The complete code for this tutorial is in [this Github repo](https://github.com/anthonygore/vue-single-file-js-components) if you want to download it.

Change into the directory and create the files we'll need:

```
$ cd sfc-simple
$ touch app.js
$ touch SingleFileComponent.js
```

Remove the inline script from *index.html* and instead use script tags to link to our modules. Note the `type="module"` attribute:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Vue.js Single-File JavaScript Component Demo</title>
  <script src="https://unpkg.com/vue"></script>
</head>
<body>
  <div id="app"></div>
  <script type="module" src="SingleFileComponent.js"></script>
  <script type="module" src="app.js"></script>
</body>
</html>
```

## Creating a single-file JavaScript component

This is a component like any other you've created, only you export the configuration object since it's a module:

*SingleFileComponent.js*

```js
export default {
  template: `
    <div>
     <h1>Single-file JavaScript Component</h1>
     <p>{{ message }}</p>
    </div>
  `,
  data() {
    return {
      message: 'Oh hai from the component'
    }
  }
}
```

Now we can import it and use it in our Vue app:

*app.js*

```js
import SingleFileComponent from 'SingleFileComponent.js';

new Vue({
  el: '#app',
  components: {
    SingleFileComponent
  }
});
```

*index.html*

```html
<div id="app">
  <single-file-component></single-file-component>
</div>
```

## Serving the app

For a simple project like this all you need is a static server on the command line with the `http-server` module:

```bash
# This will serve the project directory at localhost:8080
$ http-server
```

To view the app you will, of course, need to use a browser which supports JavaScript modules. I'm using Chrome 61.

![Project running in the browser. Shows Vue Devtools.](/images/posts/single_file_js_component_01.png)

## Fallback 

What if the user's browser doesn't support JavaScript modules? This will be the case for most users, for a while.

We can use a script tag with the `nomodule` attribute to write a simple error message to the document:

```html
<body>
  <div id="app">
    <single-file-component></single-file-component>
  </div>
  <script type="module" src="SingleFileComponent.js"></script>
  <script type="module" src="app.js"></script>
  <script nomodule>
    document.getElementById("app").innerHTML = "Your browser doesn't support JavaScript modules :(";
  </script>
</body>
```

A far better fallback, though, would be to use a Webpack bundled version of the project. This simple config will do the job:

```js
var path = require('path')
var webpack = require('webpack')

module.exports = {
  entry: './app.js',
  output: {
    path: path.resolve(__dirname, './dist'),
    publicPath: '/dist/',
    filename: 'build.js'
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        loader: 'babel-loader',
        exclude: /node_modules/
      }
    ]
  }
}
```

After a build, the bundle can now be loaded as the fallback script:

```html
<body>
  ...
  <script type="module" src="SingleFileComponent.js"></script>
  <script type="module" src="app.js"></script>
  <script nomodule src="/dist/build.js"></script>
</body>
```

This Webpack version will work identically in a browser without native module support. Here it is in Firefox, note that *build.js* has loaded and not the module:

![Project running in Firefox](/images/posts/single_file_js_component_02.png)

## Performance comparison

Since we've now got two versions of the app available, one using the native JavaScript module system, and the other using Webpack, what performance difference is there?

| | Size | Time to first meaningful paint |
| - | - | - |
| JavaScript modules | 80.7 KB | 2460 ms |
| Webpack | 83.7 KB | 2190 ms |


Using the module system gives you a smaller project size. However, the Webpack project loads quicker overall. 

> Note: these figures are from a Lighthouse test with an HTTP/2 server.

I suspect preloading would improve the speed of the modules project, but we're a bit early for this to work:

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Blink will be addressing this with &lt;link rel=modulepreload&gt; initially however this is not yet implemented.</p>&mdash; Addy Osmani (@addyosmani) <a href="https://twitter.com/addyosmani/status/908518983367745536">September 15, 2017</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

Webpack is still the better choice for module-based architectures, but it's nice to know native modules are a thing.

> *Get the latest Vue.js articles, tutorials and cool projects in your inbox with the [Vue.js Developers Newsletter](https://vuejsdevelopers.com/newsletter/?jsdojo_id=cjs_sjc)*
