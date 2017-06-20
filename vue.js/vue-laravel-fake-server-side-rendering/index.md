# Faking Server-Side Rendering With Vue.js and Laravel

Server-side rendering (SSR) is a design concept for full-stack web apps that provides a rendered page to the browser. The idea is that the page can be shown while the user waits for scripts to be downloaded and run.

If you aren't using a Node.js server for your app then you're out of luck; only a Javascript server can render a Javascript app.

However, there are alternative to SSR that may be good enough, or even better, for some use cases. In this article I'm going to explain a method I use to "fake" server-side rendering using Vue.js and Laravel.

> *Note: this article was originally posted [here on the Vue.js Developers blog](http://vuejsdevelopers.com/2017/04/09/vue-laravel-fake-server-side-rendering/?jsdojo_id=cjs_fak) on 2017/04/09*

## Pre-rendering

Pre-rendering (PR) tries to achieve the same outcome as SSR by using a headless browser to render the app and capture the output to an HTML file, which is then served to the browser. The differnce between this and SSR is that it is done ahead of time, not on-the-fly.

### Limitation: user-specific content

Some pages, like the front page of your site, will probably contain general content i.e. content that all users will view the same. But other pages, like admin pages, will contain user-specific content, for example a user's name and birth date.

The limitation of PR is that it can't be used for pages that contain such content. As I just said, the pre-rendered templates are only made once and can't be customised. SSR does not have this limitation.

## Faking server-side rendering

My fake SSR method for Vue and Laravel is to pre-render a page, but replace any user-specific content with Laravel Blade tokens. When the page is served, Laravel's `view` helper will replace the tokens with user-specific content.

So before pre-rendering your page will have this:

```html
<div id="app"></div>
```

After pre-rendering you'll have this:

```html
<div id="app">
    <div>
        Hello {{ $name }}, your birthday is {{ $birthday }}
    </div>
</div>
```

And when the page is served by Laravel your browser receives the following, which is exactly what it would receive from SSR:

```html
<div id="app" server-rendered="true">
    <div>
        Hello Anthony, your birthday is 25th October.
    </div>
</div>
```

With this method we get all the benefits of SSR but it can be done with a non-Node backend like Laravel. 

<!--more-->

## How it's done

I've setup [this repo](https://github.com/anthonygore/fake-ssr-vue-laravel) with a demo for you to refer to, but below I'll cover the main steps in getting this to work.

### 1. Vue.js app

Any user-specific content will need to be in a data property. We'll use a Vuex store to make this easier:

```js
const store = new Vuex.Store({
  state: {
    // These are the user-specific content properties
    name: null,
    birthday: null
  }
});

new Vue({
  el: '#app',
  store
});
```

When the app is being pre-rendered we want to set the user-specific data as strings containing Laravel Blade tokens. To do this we'll use the Vuex `replaceState` method after the store is created, but before the app is mounted (we'll set the value of the global `window.__SERVER__` shortly).

```js
if (window.__SERVER__) {
  store.replaceState({
    name: '{{ $name }}',
    birthday: '{{ $birthday }}'
  });
}
```

#### Client-side hydration

When the Vue app mounts we want it to take over the page. It's going to need the actual initial store state to do this, so let's provide it now rather than using AJAX. To do this we'll put the initial state in a JSON-encoded string, which we'll create in the next step. For now, let's just create the logic by modifying the above to:

```js
if (window.__SERVER__) {
  store.replaceState({
    name: '{{ $name }}',
    birthday: '{{ $birthday }}'
  });
} else {
  store.replaceState(JSON.parse(window.__INITIAL_STATE__));
}
```

### 2. Blade template

Let's set up a Blade template including:

- A mount element for our Vue app
- Inline scripts to set the global variables discussed in the previous step
- Our Webpack build script

```html
<div id="app"></div>
<script>window.__SERVER__=true</script>
<script>window.__INITIAL_STATE__='{!! json_encode($initial_state) !!}'</script>
<script src="/js/app.js"></script>
```

The value of `$initial_state` will be set by Laravel when the page is served.

### 3. Webpack configuration

We'll use the Webpack `prerender-spa-plugin` to do the pre-rendering. I've done a more detailed write up [here](/2017/04/01/vue-js-prerendering-node-laravel/) about how this works, but here's the concept in brief:

1. Put a copy of the template in the Webpack build output using `html-webpack-plugin`.
1. The `prerender-spa-plugin` will bootstrap PhantomJS, run our app, and overwrite the template copy with pre-rendered markup.
1. Laravel will use this pre-rendered template as a view.

```js
if (isProduction) {
  var HtmlWebpackPlugin = require('html-webpack-plugin');

  module.exports.plugins.push(
    new HtmlWebpackPlugin({
      template: Mix.Paths.root('resources/views/index.blade.php'),
      inject: false
    })
  );

  var PrerenderSpaPlugin = require('prerender-spa-plugin');

  module.exports.plugins.push(
    new PrerenderSpaPlugin(
      Mix.output().path,
      [ '/' ]
    )
  ); 
}
```

### 4. Post-build script

If you were to run Webpack now you'll have `index.blade.php` in your Webpack build folder and it will contain:

```html
<div id="app">
    <div>
        Hello {{ $name }}, your birthday is {{ $birthday }}
    </div>
</div>
<script>window.__SERVER__=true</script>
<script>window.__INITIAL_STATE__='{!! json_encode($initial_state) !!}'</script>
<script src="/js/app.js"></script>
```

There's some additional tasks we need to do before this can be used:

1. Add the attribute `server-rendered="true"` to the mount element. This lets Vue know that we've already rendered the page and it will attempt a seamless takeover. The `replace` NPM module can do this job.
1. Change `window.__SERVER__=true` to `window.__SERVER__=false` so that when the app runs in the browser it loads the store with the initial state.
1. Move this file to somewhere that your route can use it. Let's create a directory `resources/views/rendered` for this. (Might also be a good idea to add this to `.gitignore` just as you would for your Webpack build.)

We'll create a bash script `render.sh` to do all this:

```bash
#!/usr/bin/env bash
npm run production &&
mkdir -p resources/views/rendered
./node_modules/.bin/replace "<div id=\"app\">" "<div id=\"app\" server-rendered=\"true\">" public/index.html
./node_modules/.bin/replace "<script>window.__SERVER__=true</script>" "<script>window.__SERVER__=false</script>" public/index.html &&
mv public/index.html resources/views/rendered/index.blade.php
```

Now we can render or re-render our template anytime like this:

```bash
$ source ./render.sh
```

### 5. Route

The last step is to get our route in `web.php` to serve the pre-rendered template and use the `view` helper to replace the tokens with the user-specific data:

```php
Route::get('/', function () {
    $initial_state = [
        'name' => 'Anthony',
        'birthday' => '25th October'
    ];
    $initial_state['initial_state'] = $initial_state;
    return view('rendered.index', $initial_state);
});
```

The array `$initial_state` contains the user-specific data, though in a real app you'd probably first check that the user is authorised and grab the data from a database.

## Performance advantage of the fake SSR approach

The normal approach to displaying a page with user-specific content in a frontend app, for example the one explained in [Build an App with Vue.js: From Authentication to Calling an API](https://auth0.com/blog/build-an-app-with-vuejs/), requires some back-and-forth between the browser and server before it can actually display anything:

1. Browser requests page
2. Empty page is served and nothing is yet displayed
3. Browser requests script
4. Script now runs, does an AJAX request to server to get user-specific content
5. Content is returned so now page finally has what it needs to display something

With this approach not only can we display something much earlier, we can also eliminate an unnecessary HTTP request: 

1. Browser requests page
2. Complete page is supplied so browser can display it straight away
3. Browser requests script
4. Script now runs, has all necessary content to seamlessly take over page.

This, of course, is the advantage that real SSR has as well, the difference being that this approach makes it achievable with a non-Node.js server like Laravel! 

## Limitations

- This is a fairly fragile and complicated setup. To be fair, setting up SSR is no walk in the park either.
- Your webpack build will take longer.
- If your data undergoes manipulation by Javascript before it's displayed you have to recreate that manipulation server-side as well, in a different language. That will suck.

> *Get the latest Vue.js articles, tutorials and cool projects in your inbox with the [Vue.js Developers Newsletter](http://vuejsdevelopers.com/newsletter/?jsdojo_id=cjs_fak)*
