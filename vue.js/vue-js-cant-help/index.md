# When VueJS Can't Help You

If you want to build a web page with JavaScript, VueJS can do one helluva job on it. But there's a condition: it only works on parts of the page where it has unhampered control. Any part that might be interfered with by other scripts or plugins is a no-go for Vue.

This means the `head` and `body` tags are Vue-free zones. It's a real bummer if you wanted Vue to manage a class on the `body`, to take one example. 

But while Vue can't *directly* manage the `head` or `body` tags, it can still help you to manage them through other means.

> *Note: this article was originally posted [here on the Vue.js Developers blog](http://vuejsdevelopers.com/2017/05/01/vue-js-cant-help-head-body/?jsdojo_id=cjs_vch) on 2017/05/01*

## Vue's beef with the `head` and `body` tags

Why is Vue picky about where it works?

Vue optimises page rendering through use of a *virtual DOM*. This is a JavaScript representation of the "real" DOM that Vue keeps in memory. DOM updates are often slow, so changes are made first to the virtual DOM, allowing Vue to optimise how it updates the real DOM through batching etc.

This system would be undermined if some third party were to make changes to the DOM without Vue's knowledge causing a mismatch between the real DOM and the virtual DOM.

For this reason Vue will not attempt to control the *whole* page, but only a part of the page where it knows it will have unhampered control.

## The mount element

The first thing we usually do in a Vue project is to give Vue a mount element in the configuration object via the `el` property:

```js
new Vue({
  el: '#app'
});
```
<!--more-->
This tells Vue where we've set aside part of the page that it can have to itself. Vue will have dominion over this element and all of it's children. But it is unable to affect any element *outside* of the mount element, be it sibling or ancestor:

```html
<head>
  <!--Vue has no power here!-->
</head>
<body>
  <!--Vue has no power here!-->
  <div id="app">
    <!--Vue's dominion-->
  </div>
  <div id="not-the-app">
    <!--Vue has no power here!-->
  </div>
</body>
```

## No mounting to the `body`

You'd be excused for thinking that the `body` tag would be a better place to mount, since there are many good reasons to want to have control over `body` classes, body events etc.

The problem is that there are browser plugins and third party scripts that pollute the `body` with their own classes, event listeners and will even append their own child nodes willy-nilly.

That is just too scary for Vue, so the `body` tag is out of bounds. In fact, as of version 2, if you attempt to mount there you will get this warning: 

```
"Do not mount Vue to <html> or <body> - mount to normal elements instead."
```

## Managing the `head` and `body` tags

So now that we've established that Vue must mount on its very own node below the `body`, and it can't affect any part of the DOM above this mount node, how do you manage the `body` or `head` with Vue?

The answer is: you can't. Well not directly, at least. Anything outside the mount element is effectively invisible to Vue.

But there's more to Vue than rendering. So even though there are elements beyond it's reach, it can still assist you to reach them in other ways via watchers and lifecycle hooks.

## Scenario #1: Listening to key events

Let's say you're creating a modal window with Vue and you want the user to be able to close the window with the *escape* key. 

Vue gives you the `v-on` directive for listening to events, but unless you are focused on a form input, key events are dispatched from the `body` tag:

![](http://d33wubrfki0l68.cloudfront.net/ee71695972c42ef7d405f8ab6c9db0d44dc6b258/de8ce/images/posts/cant_help_1.png)

Since the `body` is out of Vue's jurisdiction, you won't be able to get Vue to listen to this event. You'll have to set up your own event listener with the Web API:

```js
var app = new Vue({ 
  el: '#app',
  data: {
    modalOpen: false
  }
});

document.addEventListener('keyup', function(evt) {
  if (evt.keyCode === 27 && app.modalOpen) {
    app.modalOpen = false;
  }
});
```

### How Vue can help

Vue can help via its *lifecycle hooks*. Firstly, use the *created* hook to add the listener. This ensures that data properties you're referencing (i.e. `modalOpen`) are being observed when the callback is fired. 

Secondly, use the *destroyed* hook to remove the listener when it's no longer needed to avoid memory leaks.

```js
new Vue({
  el: '#app',
  data: {
    modalOpen: false
  },
  methods: {
    escapeKeyListener: function(evt) {
      if (evt.keyCode === 27 && this.modalOpen) {
        this.modalOpen = false;
      }
    }
  },
  created: function() {
    document.addEventListener('keyup', this.escapeKeyListener);
  },
  destroyed: function() {
    document.removeEventListener('keyup', this.escapeKeyListener);
  },
});
```

## Scenario #2: Managing `body` classes

When a user opens your modal window, you want to completely disable the main window. To do this you can stack it behind a semi-transparent panel so it can't be clicked, and clip any overflow so it can't be scrolled.

![](http://d33wubrfki0l68.cloudfront.net/c78cd124d9d97dfaf9ab01cb5870a32ecddec52c/961f0/images/posts/cant_help_2.png)

To prevent scrolling, add a class to the body (let's call it `modal-open`) which makes the `overflow: hidden`.

```css
body.modal-open {
  overflow: hidden;
}
```

Obviously we need to dynamically add and remove this class, as we'll still want to allow scrolling when the modal is closed. We'd normally use `v-bind:class` to do this job, but again, you can't bind to `body` attributes with Vue, so we're going to have to use the Web API again:

```js
// Modal opens
document.body.classList.add('modal-open');

// Modal closes
document.body.classList.remove('modal-closed');
```

### How Vue can help

Vue adds reactive getters and setters to each data property so that when data value change it knows to update the DOM. Vue allows you to write custom logic that hooks into reactive data changes via *watchers*.

Vue will execute any watcher callbacks whenever the data value (in this case `modalOpen`) changes. We'll utilise this callback to update to add or remove the `body` class:

```js
var app = new Vue({
  el: '#app',
  data: {
    modalOpen: false
  },
  watch: {
    modalOpen: function(newVal) {
      var className = 'modal-open';
      if (newVal) {
        document.body.classList.add(className);
      } else {
        document.body.classList.remove(className);
      }
    }
  }
});
```

> *Get the latest Vue.js articles, tutorials and cool projects in your inbox with the [Vue.js Developers Newsletter](http://vuejsdevelopers.com/newsletter/?jsdojo_id=cjs_vch)*

