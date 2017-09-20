It's a common practice for a Vue app to use the DOM as its template, as it's the quickest and easiest architecture to set up.

This practice comes with a few catches, however, that make it an undesirable choice for any serious project. For example, the markup you write for a DOM template is not always what you get when your app runs.

In this article, I'll explain the issues with using the DOM as a template and offer some alternatives.

> *Note: this article was originally posted [here on the Vue.js Developers blog](https://vuejsdevelopers.com/2017/09/17/vue-js-avoid-dom-templates/?jsdojo_id=cjs_adt) on 2017/09/17*

## DOM as a template

The `el` option is used to mount a Vue instance to an element in the DOM. If no `template` or `render` option is present, Vue will use any existing content within the mounting element as the template.

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title>title</title>
  </head>
  <body>
    <div id="app">
      <!--This markup will be the template of the root instance-->
      <h1>My Vue.js App</h1>
      <p>{{ message }}</p>
    </div>
  </body>
</html>
```
```js
new Vue({
  el: '#app',
  data: {
    message: 'Hello world'
  }
});
```

This approach gets you up-and-running quickly, but you should transition away from it because:

- The markup you write in not always what you get
- Syntax clashes with templating engines
- Incompatibility with server-side rendering
- Runtime template compilation is required

### Markup != DOM

Let's make a distinction between *markup* and the *DOM*. Markup is the HTML you write. The browser will then parse that and turn it into the DOM.

Vue uses the DOM as a template, not the markup you write. Why is that a problem? The DOM parser and Vue do not always agree on what is acceptable markup. Non-standard markup may be changed or removed from the page, causing unpredictable results.

For example, you may have a custom component `my-row` that renders as a `tr` and would be logically used like this:

```html
<table>
  <my-row></my-row>
</table>
```

Since only `tr` is allowed inside a `table`, the browser will hoist the `my-row` element above the table during its parsing of the document. By the time Vue runs you'll have this:

```html
<my-row></my-row>
<table></table>
```

There are also cool non-standard features for Vue templates that the DOM parser will strip out:

**Self-closing tags:** if your component doesn't need a slot, Vue allows you to make it a self-closing element.

```html
<!--Will be discarded by the DOM parser-->
<my-component/>
```

**Non-kebab-cased components:** HTML is case insensitive which means you're restricted to kebab-case components in the template. Vue will happily accept camelCase or PascalCase, though. Same with props.

```html
<!--Will be discarded by the DOM parser-->
<PascalCaseComponent camelCaseProp="test"></PascalCaseComponent>
```

None of these issues occur if you use string templates (or render functions) since the DOM parser is not a factor there.

### Clash with templating engines

If you're using Vue.js in conjunction with a templating engine, any common syntax can be problematic.

For example, Handlebars and Laravel Blade both use the double curly braces `{{ }}` syntax that Vue uses. When your template is processed, the templating engine won't be able to distinguish the Vue syntax which will cause a problem.

This is usually easy to get around by escaping the Vue syntax e.g. in Blade you can put an `@` in front of your braces and Blade will know to ignore them e.g. `@{{ forVue }}`. But still, it's an additional thing to annoy you.

### Server-side rendering

You quite simply can't use a DOM template if you want to server-side render your Vue app, as the HTML document is not an input of the SSR process.

## Avoiding DOM templates

How can you architect a Vue.js app without a DOM template, or at least a small one?

### 1. Abstract markup to components

Your root instance can hold some state, but generally, you want any presentational logic and markup to be abstracted to components so it's out of your DOM template.

Single-file components are the superior choice. If you're unable to include a build step in your project and don't like writing your templates as JavaScript strings (who does), you can try *x-templates*. 

#### x-templates

With x-templates, your template is still defined in the page, but within a script tag, and will, therefore, avoid processing by the DOM parser. The script tag is marked with `text/x-template` and referenced by an `id` in your component definition.

```js
Vue.component('my-component', {
  template: '#my-component'
}
```

```html
<script type="text/x-template" id="my-component">
  <div>My component template</div>
  <NonStandardMarkupIsFineHere/>
</script>
```

### 2. Mount to an empty node with a render function

Abstracting markup into components hits a wall when you realize you still need to declare your root-level component in the DOM template. 

```html
<div id="app">
  <!-- We still have a DOM template :( -->
  <app></app>
</div>
```

If you want to totally eliminate your DOM template, you can mount your root-level component(s) with a render function.

Let's say you have one all-encompassing component that declares the other components called `App`. `App` can be declared with a `render` function and mounted to an empty node since render functions will *replace* their mount element.

```html
<div id="app"></div>
```

```js
new Vue({
  el: '#app',
  components: {
    App
  },
  render: function(createElement) {
    return createElement(App);
  }
})
```

And with that, your app is free of any DOM template!

> If you can eliminate all string and DOM templates from your app you can use the smaller runtime-only build of Vue. This is an ideal project architecture and is the one you'll see used in vue-cli templates.

## Summary

- DOM templates are problematic as the DOM parser can mess with your markup. There's also a potential for clashes with templating engines and incompatibility with server-side rendering.
- To minimize your DOM template, abstract your markup into components.
- To completely eliminate your DOM template you'll need to mount your root-level component with a render function.

> *Get the latest Vue.js articles, tutorials and cool projects in your inbox with the [Vue.js Developers Newsletter](https://vuejsdevelopers.com/newsletter/?jsdojo_id=cjs_adt)*
