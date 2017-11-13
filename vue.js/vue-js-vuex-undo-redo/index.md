There are many benefits to centralizing your application state in a Vuex store. One benefit is that all transaction are recorded. This allows for handy features like *time-travel debugging* where you can jump between previous states to isolate problems.

In this article, I'll demonstrate how to create an undo/redo feature with Vuex, which works in a similar way to time-travel debugging. This feature could be used in a variety of scenarios from complex forms to browser-based games.

You can check out the completed code [here on Github](https://github.com/anthonygore/vuex-undo-redo), and try a demo in this [Codepen](https://codepen.io/anthonygore/pen/NwGmqJ).


I've also created the plugin as an NPM module called [vuex-undo-redo](https://github.com/anthonygore/vuex-undo-redo) if you'd like to use it in a project.

> *Note: this article was originally posted [here on the Vue.js Developers blog](https://vuejsdevelopers.com/2017/11/13/vue-js-vuex-undo-redo/?jsdojo_id=cjs_vur) on 2017/11/13*

## Setting up a plugin

To make this feature reusable we'll create it as a Vue plugin. This feature requires us to add some methods and data to the Vue instance, so we'll structure the plugin as a mixin. 

*plugin.js*

```js
module.exports = {
  install(Vue) {
    Vue.mixin({
      // Code goes here
    });
  }
};
```

To use it in a project we can simply import the plugin and install it:

*app.js*

```js
import VuexUndoRedo from './plugin.js';
Vue.use(VuexUndoRedo);
```


## Concept

The feature will work by rolling back the last mutation if the user wants to undo, then re-applying it if they want to redo. How will we implement this?

### Approach #1

The first possible approach is to "snapshot" the state of the store after every mutation and pushing the snapshot into an array. To undo/redo we can grab the correct snapshot and replace the store state with it. 

The issue with this approach is that the store state is a JavaScript object. When you push a JavaScript object to an array you're just pushing a reference to the object. A naive implementation, like the following, would not work:

```js
var state = { ... };
var snapshot = [];

// Push the first state
snapshot.push(state);

// Push the second state
state.val = "new val";
snapshot.push(state);

// Both snapshots are simply a reference to state
console.log(snapshot[0] === snapshot[1]); // true
```

The snapshot approach would require that you first make a clone of the state before pushing. Given that Vue state is made reactive through the automatic additions of getter and setter functions, it doesn't play nicely with cloning. 

### Approach #2

Another possible approach is to log every mutation that is committed. To undo, we reset the store to its initial state and then re-run the mutations again; all but the last. Redoing is a similar concept. 

Given the principles of Flux, re-running the mutations from the same initial state should recreate the state perfectly. Since this is a cleaner approach than the first, let's proceed with it.

## Logging mutations

Vuex offers an API method for subscribing to mutations which we can use to log them. We'll set this up in the `created` hook. In the callback, we simply push the mutation into an array which can later be re-run.

*plugin.js*

```js
Vue.mixin({
  data() {
    return {
      done: []
    }
  },
  created() {
    this.$store.subscribe(mutation => {
      this.done.push(mutation);
    }
  }
});
```

## Undo method

To undo a mutation we will clear the store then re-run all the mutations except for the last one. Here's how the code works: 

1. Use the `pop` array method to remove the last mutation 
1. Clear the store state with a special mutation `EMPTY_STATE` (explained below)
1. Iterate each remaining mutation, committing it again to the fresh store. Note that the subscribe method is still active during this process, meaning each mutation will keep being re-added. Remove it again immediately with `pop` to prevent this. 

```js
const EMPTY_STATE = 'emptyState';

Vue.mixin({
  data() { ... },
  created() { ... },
  methods() {
    undo() {
      this.done.pop();
      this.$store.commit(EMPTY_STATE);
      this.done.forEach(mutation => {
        this.$store.commit(`${mutation.type}`, mutation.payload);
        this.done.pop();
      });
    }
  }
});
```

## Clearing the store

Whenever this plugin is used the developer must implement a mutation in their store called `emptyState`. This has the job of reverting the store back to its original state so it's ready to be re-built from scratch.

The developer must do this themselves because the plugin we're building doesn't have access to the store, only the Vue instance. Here's an example implementation:

*store.js*

```js
new Vuex.Store({
  state: {
    myVal: null
  },
  mutations: {
    emptyState() {
      this.replaceState({ myval: null });       
    }
  }
});
```

Going back to our plugin, the `emptyState` mutation should not be added to our `done` list, as we don't want to re-commit that in the undo process. Prevent this with the following logic:

*plugin.js*

```js
Vue.mixin({
  data() { ... },
  created() {
    this.$store.subscribe(mutation => {
      if (mutation.type !== EMPTY_STATE) {
        this.done.push(mutation);
      }
    });
  },
  methods() { ... }
});
```

## Redo method

Let's create a new data property `undone` which will be an array. When we remove the last mutation from `done` during the undo process, we push it to this array:

*plugin.js*

```js
Vue.mixin({
  data() {
    return {
      done: [],
      undone: []
    }
  },
  methods: {
    undo() {
      this.undone.push(this.done.pop());
      ...
    }
  }
});
```

We can now create a `redo` method which will simply take the last mutation added to `undone` and re-commit it.

*plugins.js*

```js
methods: {
  undo() { ... },
  redo() {
    let commit = this.undone.pop();
    this.$store.commit(`${commit.type}`, commit.payload);
  }
}
```

## Redo invalidation

If the user triggers an undo one or more times, then makes a fresh new commit, the contents of `undone` will be invalidated. If this happens we should empty `undone`.

We can detect new commits from within our subscribe callback when a commit is added. The logic is tricky, though, as the callback doesn't have any obvious way of knowing what is a new commit, and what is an undo/redo commit. 

The simplest approach is to set a flag `newMutation` in the instance. This will be true by default, but the undo and redo methods will temporarily set it to false. If it is set to true when a mutation is committed, the `subscribe` callback will clear the `undone` array.

*plugin.js*

```js
module.exports = {
  install(Vue) {
    Vue.mixin({
      data() {
        return {
          done: [],
          undone: [],
          newMutation: true
        };
      },
      created() {
        this.$store.subscribe(mutation => {
          if (mutation.type !== EMPTY_STATE) {
            this.done.push(mutation);
          }
          if (this.newMutation) {
            this.undone = [];
          }
        });
      },
      methods: {
        redo() {
          let commit = this.undone.pop();
          this.newMutation = false;
          this.$store.commit(`${commit.type}`, commit.payload);
          this.newMutation = true;
        },
        undo() {
          this.undone.push(this.done.pop());
          this.newMutation = false;
          this.$store.commit(EMPTY_STATE);
          this.done.forEach(mutation => {
            this.$store.commit(`${mutation.type}`, mutation.payload);
            this.done.pop();
          });
          this.newMutation = true;
        }
      }
    });
  },
}
```

The main functionality is now complete! Add the plugin to your own project, or to [my demo](https://github.com/anthonygore/vuex-undo-redo-example) to test it out.

## Public API

You'll notice in my demo that the undo and redo buttons are disabled whenever their functionality is not currently allowed. For example, if there haven't been any commits yet, you obviously can't undo or redo. A developer using this plugin may want to implement similar functionality.

To allow this, the plugin can provide two computed properties `canUndo` and `canRedo` as part of a public API. These are trivial to implement:

*plugin.js*

```js
module.exports = {
  install(Vue) {
    Vue.mixin({
      data() { ... },
      created() { ... },
      methods: { ... },
      computed: {},
      computed: {
        canRedo() {
          return this.undone.length;
        },
        canUndo() {
          return this.done.length;
        }
      },
    });
  },
}
```


> *Get the latest Vue.js articles, tutorials and cool projects in your inbox with the [Vue.js Developers Newsletter](https://vuejsdevelopers.com/newsletter/?jsdojo_id=cjs_vur)*
