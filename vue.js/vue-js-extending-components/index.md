# Extending VueJS Components

Do you have components in your Vue app that share similar options, or even template markup? 

It'd be a good idea to create a *base component* with the common options and markup, and then extend the base component to create *sub components*. Such an architecture would help you apply the DRY principle in your code (Don't Repeat Youself) which can make your code more readable and reduce the possibility of bugs.

Vue provides some functionality to help with component inheritance, but you'll also have to add some of your own ingenuity.

> *Note: this article was originally posted [here on the Vue.js Developers blog](http://vuejsdevelopers.com/2017/06/11/vue-js-extending-components/?jsdojo_id=cjs_evc) on 2017/06/11*

## Example: survey questions

Here is a simple survey made with Vue.js:

![](http://vuejsdevelopers.com/images/posts/extending_components_1.png)

You'll notice that each question has a different associated input type: 

1. Text input
2. Select input
3. Radio input

<!--more-->

A good architecture would be to make each question/input type into a different, reusable component. I've named them to correspond to the above:

1. `SurveyInputText`
2. `SurveyInputSelect`
3. `SurveyInputRadio` 

It makes sense that each question/input is a different component because each needs its own markup (e.g. `<input type="text">` vs `<input type="radio">`) and each needs its own props, methods, etc., as well. However, these components will have a lot in common:

- A question
- A validation function
- An error state

Etc. So I think this is a great use case for extending components!

## Base component

Each of the sub components will inherit from a single file component called `SurveyInputBase`. Notice the following:

- The `question` prop is going to be common across each component. We could add more common options, but let's stick with just one for this simple example.
- We somehow need to copy the props from this component to any extending component.
- We somehow need to insert the markup for the different inputs *inside* the template.

*SurveyInputBase.vue*

```html
<template>
  <div class="survey-base">
    <h4>{% raw %}{{ question }}{% endraw %}</h4>
    <!--the appropriate input should go here-->
  </div>
</template>
<script>
  export default {
    props: [ 'question' ],
  }
</script>
```

## Inheriting component options

Forgetting the template for a moment, how do we get each sub component to inherit props? Each will need `question` as a prop, as well as their own unique props:

![](http://vuejsdevelopers.com/images/posts/extending_components_2.png)

This can be achieved by importing the base component and pointing to it with the `extends` option:

*SurveyInputText.vue*

```html
<template>
  <!--How to include the question here???-->
  <input :placeholder="placeholder">
</template>
<script>
  import SurveyInputBase from './SurveyInputBase.vue';
  export default {
    extends: SurveyInputBase,
    props: [ 'placeholder' ],
  }
</script>
```

Looking in Vue Devtools, we can see that using `extends` has indeed given us the base props in our sub component:

![](http://vuejsdevelopers.com/images/posts/extending_components_3.png)

### Merge strategy

You may be wondering how the sub component inherited the `question` prop instead of overwriting it. The `extends` option implements a *merge strategy* that will ensure your options are combined correctly. For example, if the props have different names, they will obviously both be included, but if they have the same name, Vue will preference the sub component.

The merge strategy also works with other options like methods, computed properties and lifecycle hooks, combining them with similar logic. Check [the docs](https://vuejs.org/v2/guide/mixins.html#Option-Merging) for the exact logic on how Vue does it, but if you need you can define your own custom strategy.

> Note: there is also the option of using the `mixin` property in a component instead of `extends`. I prefer `extends` for this use case, though, as it has a slightly different merge strategy that gives the sub components options higher priority.

## Extending the template

It's fairly simple to extend a component's options - what about the template, though? 

The merge strategy does not work with the `template` option. We either inherit the base template, or we define a new one and overwrite it. But how can we *combine* them?

My solution to this is to use the [Pug](https://pugjs.org) pre-processor. It comes with `include` and `extends` options so it seems like a good fit for this design pattern.

### Base component

Firstly, let's convert our base component's template to Pug syntax:

```html
<template lang="pug">
  div.survey-base
    h4 {{ question }}
    block input
</template>
```

Notice the following:

- We add `lang="pug"` to our template tag to tell *vue-loader* to process this as a Pug template (also, don't forget to add the Pug module to your project as well `npm i --save-dev pug`)
- We use `block input` to declare an outlet for sub component content.

So here's where it gets slightly messy. If we want our child components to extend this template we need to put it into it's own file:

*SurveyInputBase.pug*

```pug
div.survey-base
  h4 {% raw %}{{ question }}{% endraw %}
  block input
```

We then `include` this file in our base component so it can still be used as a normal standalone component:

*SurveyInputBase.vue*

```html
<template lang="pug">
  include SurveyInputBase.pug
</template>
<script>
  export default {
    props: [ 'question' ]
  }
</script>
```

It's a shame to have to do that since it kind of defeats the purpose of "single file" components, but for this use case, I think it's worth it. Probably you could make a custom webpack loader to avoid having to do this.

### Sub component

Let's now convert our sub component's template to Pug as well:

*SurveyInputText.vue*

```html
<template lang="pug">
  extends SurveyInputBase.pug
  block input
    input(type="text" :placeholder="placeholder")
</template>
<script>
  import SurveyInputBase from './SurveyInputBase.vue';
  export default {
    extends: SurveyInputBase,
    props: [ 'placeholder' ],
  }
</script>
```

The sub components use the `extends` feature of Pug which includes the base component and outputs any custom content in the `input` block (a concept analgous to *slots*). 

Here's what the sub component's template would effectively look like after extending the base and being translated back to a regular HTML Vue template:

```html
<div class="survey-base">
  <h4>{% raw %}{{ question }}{% endraw %}</h4>
  <input type="text" :placeholder="placeholder">
</div>
```

## Bring it all together

Using this strategy we can go ahead and create the other two sub components `SurveyInputSelect` and `SurveyInputRadio` with their own props and markup.

If we then use them in a project our main template might look like this:

```html
<survey-input-text
  question="1. What is your name?"
  placeholder="e.g. John Smith"
></survey-input-text>

<survey-input-select
  question="2. What is your favorite UI framework?"
  :options="['React', 'Vue.js', 'Angular']"
></survey-input-select>

<survey-input-radio
  question="3. What backend do you use?"
  :options="['Node.js', 'Laravel', 'Ruby']"
  name="backend"
>
</survey-input-radio>
```

And here's the rendered markup:

```html
<div class="survey-base">
  <h4>1. What is your name?</h4>
  <input type="text" placeholder="e.g. John Smith">
</div>
<div class="survey-base">
  <h4>2. What is your favorite UI framework?</h4>
  <select>
    <option>React</option>
    <option>Vue.js</option>
    <option>Angular</option>
  </select>
</div>
<div class="survey-base">
  <h4>3. What backend do you use?</h4>
  <div><input type="radio" name="backend" value="Node.js">Node.js</div>
  <div><input type="radio" name="backend" value="Laravel">Laravel</div>
  <div><input type="radio" name="backend" value="Ruby">Ruby</div>
</div>
```

> Note: we could also instantiate the `SurveyInputBase` component, since it will work standalone, but it wasn't really needed in this example. I thought that was an important point to mention, though.

> *Get the latest Vue.js articles, tutorials and cool projects in your inbox with the [Vue.js Developers Newsletter](http://vuejsdevelopers.com/newsletter/?jsdojo_id=cjs_evc)*
