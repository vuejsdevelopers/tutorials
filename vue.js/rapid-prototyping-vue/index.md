# Speed Up Development With Prototyping and Vue

![](prototyping.jpg)

Have you ever worked on a project that got away from you? It's easy to start with a project, only to find yourself overwhelmed with new skills and challenges. Prototypes help keep projects small, and help developers decide if they're worth pursuing. Furthermore, we can leverage the flexibility of Vue to build prototypes, so we can iterate on ideas quickly.

*Note: this article was originally posted [here on the Vue.js Developers blog](http://vuejsdevelopers.com/2017/05/20/vue-js-safely-jquery-plugin/?jsdojo_id=cjs_sjp) on 2017/06/04

## Product vs. Prototype

A **product** is what a user sees, like an app, a service, or a device. A **prototype** resembles the product, but is meant to experiment with a particular feature of the product at a low cost.

As web developers, we often make the product before the prototype. If you've ever had a weekend project turn into a month-long effort, consider taking the prototyping approach. Prototypes have some useful benefits:

* You can make a prototype in an hour, as opposed to a day
* Prototypes are not production-focused, so you don't need a production environment (testing, deploying, etc.)
* You can prototype one feature at a time, to see how it fits into your product (and if it's worth implementing)
* If you'd like to try out some new technology, you can use a prototype to explore it, instead of experimenting in a production environment

Products take more time to make, requiring research and testing. On the other hand, prototypes require less time and fewer resources. We'll explore how to make a prototype instead of a product with modern web tools including Vue.js.

## An Example

Suppose we want to make an app that tells a user when the sun will rise and set at a given location. Furthermore, we'd like to display a visual that shows what time in the day the sun rises/sets.

> **Goal**: Make an app that shows the times that the sun will rise and set at a given location. The times should be shown in a data visualization.

Throughout this example, we'll identify **minimum viable solutions** for our prototype. These minimum viable solutions come when we have to make a decision about what we're building. We'll choose an option more appropriate for building a prototype, instead of a product.

This is what the final product will look like:



<p data-height="265" data-theme-id="0" data-slug-hash="rvqKEV" data-default-tab="html,result" data-user="pj_" data-embed-version="2" data-pen-title="Sunrise Sunset Prototype Example" class="codepen">See the Pen <a href="https://codepen.io/pj_/pen/rvqKEV/">Sunrise Sunset Prototype Example</a> by PJ Trainor (<a href="https://codepen.io/pj_">@pj_</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

> This is an actual example of an idea I had that _should_ have been a prototype, but ended up being an unfinished product. Let's see if we can learn from my mistakes!

### Choosing the Environment

There are a number of excellent resources for development. For example, we could work locally with command-line tools. This offers a fully-featured development environment, however, that's a lot of overhead for a simple prototype.

Then there's [Codesandbox](https://codesandbox.io/), which replicates a development environment in the browser. It has no setup, and it's a great option if we want to move to production.

Lastly, there's [CodePen](https://codepen.io/), which seems promising because it keeps our prototype isolated and simple. Also, the site offers many great ways to share our ideas with others.

For our purposes, we'll choose CodePen because it makes us think _small_. Having one file for HTML, CSS, and js forces us to write less code, so we don't add more features. This makes CodePen a great way to put something together _fast_, and that's why it's the _minimum viable solution_ for our development environment.

> **Note**: If you'd rather work with command-line tools, I recommend using [vue-cli](https://github.com/vuejs/vue-cli). See [this article](https://vuejsdevelopers.com/2018/03/26/vue-cli-3/), which has a section on rapid prototyping.

### Choosing a Framework

There are _many_ ways to make an app with JavaScript. We expect to render a relatively complex UI with some user interaction, so a framework like Vue, React, or Angular are the best options.

All of these frameworks are _great_ options, but we're looking for something that we can add to our app instantly, so we can get working right away.

In that case, Vue is an excellent choice. We just need to load it in a `<script>` tag, and then add `Vue({el: "#app"})` to our js file. From there, we can load HTML templates, provide our data, and render components quickly.

Vue also offers reactivity for our data. This is helpful on a small scale, where our state is quite simple, so we can incorporate user inputs seamlessly. Also, if we get stuck, we can turn to the excellent [documentation](https://vuejs.org/), available in several languages.

For writing our prototype, Vue is the _minimum viable solution_. Note that Vue makes it easy to go from prototype to product, which we will discuss briefly later on.

### Getting the Data

We need to decide what data to use. In our final product, we'll probably use an API to grab the sunset/sunrise times. Using an API would require the following:

* Finding a data source, like [this one from US Navy](http://aa.usno.navy.mil/data/docs/RS_OneYear.php)
* Parsing that data, which is in an [unusual format](http://aa.usno.navy.mil/cgi-bin/aa_rstablew.pl?ID=AA&year=2018&task=0&state=FL&place=Orlando)
* Deciding on how we'd like to access that data, perhaps using [Axios](https://github.com/axios/axios)... that actually might require some extra learning if we're not familiar with it
* Perhaps using the user's location to automatically get the sunrise/sunset times... that might require [even _more_ learning](https://developer.mozilla.org/en-US/docs/Web/API/Location)

That seems like a lot of overhead for a simple prototype. Alternatively, we could just make some dummy data. Our dummy data would look something like this:

```js
// when the sun will rise/set in different cities
const DUMMY_DATA = [
  {
    city: 'Baltimore',
    sunrise: { hour: 4, minute: 50 },
    sunset: { hour: 19, minute: 16 }
  },
  {
    city: 'Havana',
    sunrise: { hour: 5, minute: 4 },
    sunset: { hour: 19, minute: 1 }
  }
];
```

> **Note**: Dummy data just means data that you generate, to act in place of real data while we make our prototype

Remember, we're building a prototype. The API solution is the right answer for the product, but implementing it will take too long. The _minimum viable solution_ is to use dummy data.

### Creating the Visualization

Next, we need to decide how we'd like to create the sunrise/sunset visualization. We want to show the times of each event, where they happen in the day, and show when the sun is actually out.

Since we'll want our app to have a specific look, we consider making a custom component for the visualization. Making our own component would require:

* Using SVG to properly render the rectangle and text
* Figuring out how to correctly position those SVG elements
* Choosing the right colors
* Positioning it appropriately for device sizes, because we _will_ care about that when we start building

That's a lot to consider, and we'll probably have to learn a lot to get it production-ready... Alternatively, we could use an open-source component. Our design looks like a slider, and a quick search suggests [vue-slider-component](https://github.com/NightCatSama/vue-slider-component). We can even [customize it!](https://nightcatsama.github.io/vue-slider-component/example/#pretty-slider)

Here's how we might use it in Vue:

```html
<vue-slider
    :value="[sunrise, sunset]"
    :min="startOfTheDay"
    :max="endOfTheDay"
    :processStyle="{
      backgroundColor: 'darkorange'
    }"
    :bgStyle="{
      backgroundColor: 'darkslateblue'
    }"
    :sliderStyle="{
      backgroundColor: 'darkorange',
      boxShadow: 'none'
    }">
</vue-slider>
```

This produces the following:

![A slider showing sunrise/sunset times](prototyping_bar-example.png)

For prototyping purposes, the slider is perfectly suitable. Eventually, we'll make a custom component, but that requires a lot more effort for now. Using an open-source component is the _minimum viable solution_.

### Styling the App

Any app, even a prototype, should have some styling. CSS is an essential part of web development, and developers should design apps with CSS in mind.

We may choose to use a CSS specialist, or to learn more CSS on our own. This, however, adds more technology we'll have to learn. We might get carried away thinking about browser support for various features, designing for mobile devices, and how to use CSS and js together.

We know enough CSS to get by, but it will take substantial effort to make it production-ready. Alternatively, we could use open-source solutions. There are two ways we could go:

1.  Vue libraries like [Vuetify](https://vuetifyjs.com/en/) or [bootstrap-vue](https://bootstrap-vue.js.org/) give us access to beautiful components, or
2.  CSS utility libraries like [Tailwind CSS](https://tailwindcss.com/) or [Tachyons](http://tachyons.io/)

Here's an example of how to use tailwind to style a button:

![Styling a button with [tailwind](https://tailwindcss.com/docs/what-is-tailwind/#component-friendly)](prototyping_tailwind-button.png)

Solutions like tailwind easy to use with our current knowledge of CSS. Furthermore, we note that _all_ of the above options are capable of transitioning into a product when the time comes. Thus, these CSS solutions are more than suitable for prototyping purposes.

For styling our prototype, the _minimum viable solution_ is to use pre-style components or a CSS utility library. We'll use tailwind for ours.

### Building the Prototype

We have made three key decisions that will help us build a prototype quickly. We have decided to

* Develop in CodePen, to keep our ideas small
* Use Vue as our framework, for it's ease-of-use and reactivity
* Create dummy data, instead of using an API
* Use an open-source component instead of designing a custom one
* Use a CSS utility library, so we can style our app efficiently

This article won't spend any more time on building out this particular example. If you'd like to build it for yourself, see the example on CodePen. This pen has dummy data, detailed comments, and utilities loaded in.

> [Link to sunrise/sunset app example](https://codepen.io/pj_/pen/rvqKEV/).
> Try forking it!

### From Prototype to Product

After building our prototype, we can decide whether or not to continue towards production. There are several things to consider, such as

* How much time it will take to build
* How much you'll need to learn new skills
* How you plan to develop/deploy/distribute the application

Notice that when choosing our _minimum viable solutions_, we determined all of these things. We noted the difficulty in getting an API set up, and the time it might take to write a custom component. Here's a summary of the decisions for a prototype vs. a product (I've **bolded** the common solutions):

| Prototype                      | Product                                                                                                                                                    |
| :----------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CodePen for development        | Develop locally with command-line tools, or develop a more robust prototype in Codesandbox or with [**CodePen projects**](https://codepen.io/pro/projects) |
| **Write app in Vue**           | **Write app in Vue**                                                                                                                                       |
| Use dummy data                 | Get data from an API                                                                                                                                       |
| **Use open-source components** | Make a custom component, or **improve the open-source one**                                                                                                |
| Use a **CSS utility library**  | Use a **CSS utility** or component library                                                                                                                 |

If these challenges sound exciting, then go ahead--you spent some time up-front hashing out your ideas. If you don't think you can commit, then that's fine too--you only spent an hour on this.

Overall, the more you code, the more you'll learn. As you practice prototyping, you'll be able to implement new features faster. Prototyping lets you work on a project _and_ learn!

## Closing Remarks

If you have an idea for a product, it's best to start with a prototype. When developing prototypes, be mindful of the decisions that will lead you in the right direction. When prototyping, it's important to think small throughout the entire process, and to have an objective in mind. The prototyping process will help you when it comes time to make a product, and Vue is excellent for developing both.

## Resources for Rapid Prototyping

* [A Vue + Tailwind template](https://codepen.io/pj_/pen/XZpPrN) in CodePen
* [vue-cli version 3](https://vuejsdevelopers.com/2018/03/26/vue-cli-3/) provides a flexible environment to develop, test, and build your applications
* [Mockaroo](https://mockaroo.com/), a dummy-data generator
* Guides for [flexbox](https://css-tricks.com/snippets/css/a-guide-to-flexbox/) and [css-grid](https://css-tricks.com/snippets/css/complete-guide-grid/)
* Sarah Dranser's article on [Components, Slots, and Props](https://css-tricks.com/intro-to-vue-2-components-props-slots/)
* Adam Wathan's article on [renderless components](https://adamwathan.me/renderless-components-in-vuejs/) (This is an advanced, but useful topic)

*Get the latest Vue.js articles, tutorials and cool projects in your inbox with the [Vue.js Developers Newsletter](	https://vuejsdevelopers.com/newsletter/?jsdojo_id=cjs_rpv)*
