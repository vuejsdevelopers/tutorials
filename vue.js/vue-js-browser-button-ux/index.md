# Don't Forget Browser Button UX In Your Vue.js App

When building single-page applications many Vue developers forget about UX for browser button navigation. They mistakenly assume that this kind of navigtion is the same as hyperlink navigation when in fact it can be quite different. 

Unlike hyperlink navigation, if a user goes forward and back between pages they expect the page to still look like it did when they return or they'll consider the UX "weird" or "annoying".

For example, if I were browsing a thread on [Hacker News](http://news.ycombinator.com) and I scroll down to a comment and collapse it, then I clicked though to another page, then I clicked "back", I'd expect to still be scrolled down to the comment and for it to still be collapsed!

![Hacker News comments](http://vuejsdevelopers.com/images/posts/page_nav_1.png)

In a Vue.js app, though, this is not the default behaviour; scroll position and app data are not persisted by default. We need to consciously set up our app to ensure we have a smooth and predictable UX for the browser navigation buttons.

> *Note: this article was originally posted [here on the Vue.js Developers blog](http://vuejsdevelopers.com/2017/04/16/vue-js-browser-button-ux/?jsdojo_id=cjs_bbn) on 2017/04/16*

## Configuring Vue Router

Vue Router's role in optimal back and forward UX is in controlling *scroll behavior*. A user's expectations with this would be:

<!--more-->

- When moving back and forward, return to the previous scroll position
- When navigating by links, scroll to the top

We can achieve this by adding a `scrollBehavior` callback to our router configuration. Note that `savedPosition` is made available when using the browser back and forward buttons and not when using hyperlinks.

```js
const scrollBehavior = (to, from, savedPosition) => {
  if (savedPosition) {
    return savedPosition
  } else {
      position.x = 0
      position.y = 0
    }
    return position
  }
}

const router = new VueRouter({
  mode: 'history',
  scrollBehavior,
  routes: []
})
```

More comprehensive scroll behaviour settings can be found in this [example](https://github.com/vuejs/vue-router/blob/dev/examples/scroll-behavior/app.js).

## State persistence

Even more critical than scroll behaviour is persisting the state of the app. For example, if a user makes a selection on page 1, then navigates to page 2, then back to page 1, they expect the selection to be persisted. 

In the naive implementation below, *Foo*'s `checked` state will not persist between route transitions. When the route changes, Vue destroys *Foo* and replaces it with *Home*, or vice versa. As we know with components, the state is created freshly on each mount.


```js
const Foo = Vue.component('foo', {
  template: '<div @click="checked = !checked">{{ message }}</div>',
  data () {
    return { checked: false }; 
  }
  computed: {
    message() {
      return this.checked ? 'Checked' : 'Not checked';
    }
  }
});

const router = new VueRouter({
  mode: 'history',
  scrollBehavior,
  routes: [
    { path: '/', component: Home },
    { path: '/bar', component: Foo }
  ]
});
```

This would be equivalent to uncollapsing all the comments you collapsed in Hacker News when you navigate back to an article's comments i.e. very annoying!

## keep-alive

The special `keep-alive` component can be used to alleviate this problem. It tells Vue *not* to destroy any child components when they're no longer in the DOM, but instead keep them in memory. This is useful not just for a route transition, but even when `v-if` takes a component in and out of a page.

```html
<div id="app">
  <keep-alive>
    <router-view></router-view>
  </keep-alive>
</div>
```

The advantage of using `keep-alive` is that's it's very easy to setup; it can be simply wrapped around a component and it works as expected.

## Vuex

There's a scenario where `keep-alive` will not be sufficient: what if the user refreshes the page or clicks back and forward to another website? The data would be wiped and we're back to square one. A more robust solution than `keep-alive` is to use the browser's local storage to persist component state. 

Since HTML5 we can use the browser to store a small amount of arbitrary data. The easiest way to do this is to first set up a Vuex store. Any data that needs to be cached between route transitions or site visits goes in the store. Later we will persist it to local storage.

Let's now modify our example above to use Vuex to store *Foo*'s `checked` state:

```js
const store = new Vuex.Store({
  state: {
    checked: false
  },
  mutations: {
    updateChecked(state, payload) {
      state.checked = payload;
    }
  }
});

const Foo = Vue.component('foo', {
  template: '<div @click="checked">{{ message }}</div>',
  methods: {
    checked() {
      this.$store.commit('updateChecked', !this.$store.state.checked);
    }
  },
  computed: {
    message() {
      return this.$store.state.checked ? 'Checked' : 'Not checked';
    }
  }
});
```

We can now get rid of the `keep-alive`, as changing the page will no longer destroy the state information about our component as Vuex persists across routes.

## Local storage

Now, every time the Vuex store is updated, we want store a snapshot of it in local storage. Then when the app is first loaded we can check if there's any local storage and use it to seed our Vuex store. This means that even if we navigate to another URL we can persist our state.

Fortunately there's a tool for this already: [vuex-localstorage](https://github.com/crossjs/vuex-localstorage). It's really easy to setup and integrate into Vuex, below is everything you need to get it to do what was just described:

```js
import createPersist from 'vuex-localstorage';

const store = new Vuex.Store({
  plugins: [ createPersist({
    namespace: 'test-app',
    initialState: {},
    expires: 7 * 24 * 60 * 60 * 1000
  }) ],
  state: {
    checked: false
  },
  mutations: {
    updateChecked(state, payload) {
      state.checked = payload;
    }
  }
});
```

## Back and forward UX vs. hyperlink UX

You may want to differentiate behavior between back and forward navigation and hyperlink navigation. We expect data in back and forward navigation to persist, while in hyperlink navigation it should not.

For example, returning to Hacker News, a user would expect comment collapse to be reset if you navigate with hyperlinks back to the front page and then back into a thread. Try it for yourself and you'll notice this subtle difference in your expectation.

In a Vue app we can simply add a *navigation guard* to our home route where we can reset any state variables:

```js
const router = new VueRouter({
  mode: 'history',
  scrollBehavior,
  routes: [
    { path: '/', component: Home, beforeEnter(to, from, next) {
      store.state.checked = false;
      next();
    } },
    { path: '/bar', component: Foo }
  ]
});
```

> *Get the latest Vue.js articles, tutorials and cool projects in your inbox with the [Vue.js Developers Newsletter](http://vuejsdevelopers.com/newsletter/?jsdojo_id=cjs_bbn)*
