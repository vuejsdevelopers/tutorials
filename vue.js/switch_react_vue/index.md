# Switching From React To Vue.js

So you're a React developer and you've decided to try out Vue.js. Welcome to the party!

React and Vue are kind of like Coke and Pepsi, so much of what you can do in React you can also do in Vue. There are some important conceptual differences though, some of which reflect Angular's influence on Vue. 

I'll focus on the differences in this article so you're ready to jump into Vue and be productive straight away.

*Note: this article was originally posted [here on the Vue.js Developers blog](http://vuejsdevelopers.com/2017/05/28/switch-from-react-to-vue-js/?jsdojo_id=cjs_srv) on 2017/05/28*

## How much difference is there between React and Vue?

React and Vue have more similarities than differences:

- Both are JavaScript libraries for creating UIs 
- Both are fast and lightweight 
- Both have a component-based architecture
- Both use a virtual DOM
- Both can be dropped into a single HTML file or be a module in a more sophisticated Webpack setup
- Both have separate, but commonly used, router and state management libraries

The big differences are that Vue typically uses an *HTML template file* where as React is fully JavaScript. Vue also has *mutable state* and an automatic system for re-rendering called "reactivity".

We'll break it all down below.

## Components

With Vue.js, components are declared with an API method `.component` which takes arguments for an `id` and a definition object. You'll probably notice familiar aspects of Vue's components, and not-so-familiar aspects:

```js
Vue.component('my-component', {
  
  // Props
  props: [ 'myprop' ],
  
  // Local state
  data() {
    return {
      firstName: 'John',
      lastName: 'Smith'
    }
  },

  // Computed property
  computed: {
    fullName() {
      return this.firstName + ' ' + this.lastName;
    }
  },

  // Template
  template: `
    <div>
      <p>Vue components typically have string templates.</p>
      <p>Here's some local state: {{ firstName }}</p>
      <p>Here's a computed value: {{ fullName }}</p>
      <p>Here's a prop passed down from the parent: {{ myprop }}</p>
    </div>
  `,

  // Lifecycle hook
  created() {
    setTimeout(() => {
      this.message = 'Goodbye World'  
    }, 2000);
  }
});
```

### Template

You'll notice the component has a `template` property which is a string of HTML markup. The Vue library includes a compiler which turns a template string into a `render` function at runtime. These render functions are used by the virtual DOM.

You can choose *not* to use a template if you instead want to define your own `render` function. You can even use JSX. But switching to Vue just to do that would be kind of like visiting Italy and not eating pizza...

### Lifecycle hooks

Components in Vue have similar lifecycle methods to React components as well. For example, the `created` hook is triggered when the component state is ready, but before the component has been mounted in the page.

One big difference: there's no equivalent for `shouldComponentUpdate`. It's not needed because of Vue's reactivity system.

### Re-rendering

One of Vue's initialisation steps is to walk through all of the data properties and convert them to getters and setters. If you look below, you can see how the `message` data property has a get and set function added to it:

![](/images/posts/switch_react_vue_1.png)

Vue added these getters and setters to enable dependency tracking and change notification when the property is accessed or modified.

#### Mutable state

To change the state of a component in Vue you don't need a `setState` method, you just go ahead and mutate:

```js
// React
this.setState({ message: 'Hello World' });

// Vue
this.message = 'Hello World';
```

When the value of `message` is changed by the mutation, its setter is triggered. The `set` method will set the new value, but will also carry out a secondary task of informing Vue that a value has changed and any part of the page relying on it may need to be re-rendered.

If `message` is passed as a prop to any child components, Vue knows that they depend on this will be automatically re-rendered as well. That's why there's no need for a `shouldComponentUpdate` method on Vue components.

## Main template

Vue is more like Angular with regards to the main template file. As with React, Vue needs to mounted somewhere in the page:

```html
<body>
  <div id="root"></div>
</body>
```

```js
// React
ReactDOM.render('...', document.getElementById('root'));

// Vue
new Vue({
  el: '#root'
});
```

But unlike React, you can continue to add to this main `index.html` as it is the template for your root component.

```html
<div id="root">
  <div>You can add more markup to index.html</div>
  <my-component v-bind:myprop="myval"></my-component>
</div>
```

There's also a way to define your child component templates in the `index.html` as well by using HTML features like `x-template` or `inline-template`. This is not considered a best practice though as it separates the template from the rest of the component definition.

## Directives

Again, like Angular, Vue allows you to enhance your templates with logic via "directives". These are special HTML attributes with the v- prefix, e.g. `v-if` for conditional rendering and `v-bind` to bind an expression to regular HTML attribute. 

```js
new Vue({
  el: '#app',
  data: {
    mybool: true,
    myval: 'Hello World'
  }
});
```

```html
<div id="app">
  <div v-if="mybool">This renders if mybool is truthy.</div>
  <my-component v-bind:myprop="myval"></my-component>
</div>
```

The value assigned to a directive is a JavaScript expression, so you can refer to data properties, include ternary operators, etc.

## Workflow

Vue doesn't have an official `create-react-app` equivalent though there is the community built [`create-vue-app`](https://www.npmjs.com/package/create-vue-app). 

The official recommendation for bootstrapping a project, however, is `vue-cli`. It can generate anything from a simple project with one HTML file, to a fully decked-out Webpack + Server-Side Rendering project:

```bash
$ vue init template-name project-name 
```

### Single HTML file projects

Vue's creator Evan You dubbed his project a "progressive framework" because it can be scaled up for complex apps, or scaled down for simple apps.

React can do this too, of course. The difference is that Vue projects typically use less ES6 features and rarely use JSX, so there's usually no need to add Babel. Plus, the Vue library all comes in one file, there's no separate file for an equivalent of ReactDOM. 

Here's how you add Vue to a single HTML file project:

```html
<script src="https://unpkg.com/vue/dist/vue.js"></script>
```

> Note: if you don't intend to use template strings and therefore don't need the template compiler, there is a smaller build of Vue that omits this called `vue.runtime.js`. It's about 20KB smaller.

### Single file components

If you're happy to add a build step to your project with a tool like Webpack, you can utilise Vue's Single File Components (SFCs). These are files which have the `.vue` extension and encapsulate the component template, Javascript configuration and style all in a single file:

```vue
<template>
  <div class="my-class">{{ message }}</div>
</template>
<script>
  export default {
    data() {
      message: 'Hello World'
    }
  }
</script>
<style>
  .my-class { font-weight: bold; }
</style>
```

These are without a doubt one of the coolest features of Vue because you get a "proper" template with HTML markup, but the JavaScript is right there so there's no awkward separation of the template and logic.

There's a Webpack loader called `vue-loader` which takes care of processing SFCs. In the build process the template is converted to a render function so this is a perfect use cases for the the cut-down `vue.runtime.js` build in the browser.

## Redux and more

Vue also has a Flux-based state management library called Vuex. Again, it's similar to Redux, but has a number of differences.

I don't have time to cover it in this article so I'll cover it next week's article. [Join my newsletter](/newsletter) to get an email update when it's ready!

*Get the latest Vue.js articles, tutorials and cool projects in your inbox with the [Vue.js Developers Newsletter](http://vuejsdevelopers.com/newsletter/?jsdojo_id=cjs_srv)*
