# How To (Safely) Use A jQuery Plugin With Vue.js

It's not a great idea to use jQuery and Vue.js in the same UI. Don't do it if you can avoid it. 

But you're probably reading this not because you *want* to use jQuery and Vue together, but because you *have to*. Perhaps a client is insisting on using a particular jQuery plugin that you won't have time to rewrite for Vue.

If you're careful about how you do it, you can use jQuery and Vue together safely. In this article I'm going to demonstrate how to add the *jQuery UI Datepicker* plugin to a Vue project. 

And just to show off a bit, I'm even going to send data between this jQuery plugin and Vue!

See it working in this [JS Bin](https://jsbin.com/bepikonube/2/edit?html,js,output).

![jQuery UI Datepicker](jquery_plugin_1.png)

*jQuery UI Datepicker*

*Note: this article was originally posted [here on the Vue.js Developers blog](http://vuejsdevelopers.com/2017/05/20/vue-js-safely-jquery-plugin/?jsdojo_id=cjs_sjp) on 2017/05/20

## The problem with using jQuery and Vue together

Why is doing this potentially hazardous?

Vue is a jealous library in the sense that you must let it completely own the patch of DOM that you give it (defined by what you pass to `el`). If jQuery makes a change to an element that Vue is managing, say, adds a class to something, Vue won't be aware of the change and is going to go right ahead and overwrite it in the next update cycle.

## Solution: use a component as a wrapper

Knowing that Vue and jQuery are never going to share part of the DOM, we have to tell Vue to cordon off an area and give it over to jQuery.

<!--more-->

Using a component to wrap a jQuery plugin seems like the way to go because:

- We can utilise lifecycle hooks for setup and teardown of the jQuery code
- We can use the component interface to communicate with the rest of the Vue app via props and events
- Components can opt-out of updates with `v-once`

## Set up jQuery UI Datepicker

Obviously you need to include both the jQuery and jQuery UI libraries in your project first. Once you have those, the datepicker just requires an `input` element to attach itself to:

```html
Date: <input id="datepicker"/>
```

It can then be instantiated by selecting it and calling the method:

```js
$('#datepicker').datepicker();
```

## Datepicker component

To make our datepicker component, the template will be this one `input` element:

```js
Vue.component('date-picker', function() 
  template: '<input/>'
});

new Vue({
  el: '#app'
});
```

```html
<div id="app">
  Date: <date-picker></date-picker>
</div>
```

> Note: this component should be nothing more than a wrapper for the plugin. Don't push your luck and give it any data properties or use directives or slots.

### Instantiating the widget

Rather than giving our `input` an ID and selecting it, we can use `this.$el`, as every component can access its own root node like that. The root node will of course be the `input`.

We can then wrap the node reference in a jQuery selector to access the `datepicker` method i.e. `$(this.$el).datepicker()`.

Note that we use the `mounted` lifecycle hook as `this.$el` is undefined until the component is mounted.

```js
Vue.component('date-picker', function() {
  template: '<input/>',
  mounted: function() {
    $(this.$el).datepicker();
  }
});
```

### Teardown

To teardown the datepicker we can follow a similar approach and use a lifecycle hook. Note that we must use `beforeDestroy` to ensure our `input` is still in the DOM and thus can be selected (it's undefined in the `destroy` hook).

```js
Vue.component('date-picker', {
  template: '<input/>',
  mounted: function() {
    $(this.$el).datepicker();
  },
  beforeDestroy: function() {
    $(this.$el).datepicker('hide').datepicker('destroy');
  }
});
```

## Pass config with props

To make our component reusable, it would be nice to allow for custom configuration, like specifying the date format with the configuration property `dateFormat`. We can do this with `props`:

```js
Vue.component('date-picker', {
  template: '<input/>',
  props: [ 'dateFormat' ],
  mounted: function() {
    $(this.$el).datepicker({
      dateFormat: this.dateFormat
    });
  },
  beforeDestroy: function() { ... }
});
```

```html
<div id="app">
  <date-picker date-format="yy-mm-dd"></date-picker>
</div>
```

## Letting jQuery handle updates

Let's say that, rather than passing your `dateFormat` prop as a string, you made it a `data` property of your root instance i.e.:

```js
var vm = new Vue({
  data: {
    ...
    dateFormat: 'yy-mm-dd'
  }
});
```

```html
<div id="app">
  <date-picker date-format="dateFormat"></date-picker>
</div>
``` 

This would mean `dateFormat` would be a reactive data property. You could update its value at some point in the life of your app:

```js
// change the date format to something new
vm.dateFormat = 'yy-dd-mm';
```

Since the `dateFormat` prop is a dependency of the datepicker component's `mounted` hook, updating it  would trigger the component to re-render. This would not be cool. jQuery has already setup your datepicker on the `input` and is now managing it with it's own custom classes and event listeners. An update of the component would result in the `input` being replaced and thus jQuery's setup would be instantly reset.

We need to make it so that reactive data can't trigger an update in this component...

### v-once

The `v-once` directive is used to cache a component in the case that it has a lot of static content. This in effect makes the component opt-out from updates.

This is actually perfect to use on our plugin component, as it will effectively make Vue ignore it. This gives us some confidence that jQuery is going to have unhampered control over this element during the lifecycle of the app.

```html
<div id="app">
  <date-picker date-format="yy-mm-dd" v-once></date-picker>
</div>
```

## Passing data from jQuery to Vue

It'd be pretty useless to have a datepicker if we couldn't retrieve the picked date and use it somewhere else in the app. Let's make it so that after a value is picked it's printed to the page.

We'll start by giving our root instance a `date` property:

```js
new Vue({
  el: '#app',
  data: {
    date: null
  }
});
```

```html
<div id="app">
  <date-picker date-format="yy-mm-dd" v-once></date-picker>
  <p>{{ date }}</p>
</div>
```

The datepicker widget has an `onSelect` callback that is called when a date is picked. We can then use our component to emit this date via a custom event:

```js
mounted: function() {
  var self = this;
  $(this.$el).datepicker({
    dateFormat: this.dateFormat,
    onSelect: function(date) {
      self.$emit('update-date', date);
    }
  });
}
```

Our root instance can listen to the custom event and receive the new date:

```html
<div id="app">
  <date-picker @update-date="updateDate" date-format="yy-mm-dd" v-once></date-picker>
  <p>{{ date }}</p>
</div>
```

```js
new Vue({
  el: '#app',
  data: {
    date: null
  },
  methods: {
    updateDate: function(date) {
      this.date = date;
    }
  }
});
```
Thanks to [this Stack Overflow answer](http://stackoverflow.com/a/40350880/2278963) for inspiration.

*Get the latest Vue.js articles, tutorials and cool projects in your inbox with the [Vue.js Developers Newsletter](http://vuejsdevelopers.com/newsletter/?jsdojo_id=cjs_sjp)*
