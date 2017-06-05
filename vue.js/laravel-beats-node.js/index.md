# Single Page App Backends: Where Laravel Beats Node.js

I've been commissioned to write a book about building full stack Vue.js apps. Since many Laravel developers are interested in Vue (Vue now ships with Laravel), the publisher wants the book to focus on full stack Vue.js *with Laravel*.

In preparing for the book I knew I would have to answer a very important question for myself: *why would anyone even want to use Laravel as backend for a single page app when they can use Node.js?*

> *Note: this article was originally posted [here on the Vue.js Developers blog](http://vuejsdevelopers.com/2017/06/04/vue-js-backend-laravel-beats-node/?jsdojo_id=cjs_lbn) on 2017/06/04*

## Node.js advantages

Like many web devs who learned to code in the last decade, I started out with PHP. But as I got interested in frontend development and SPAs (single page apps), I eventually made the switch to full stack JavaScript and I hadn't really looked back since.

Node.js has some very clear advantages as an SPA backend:

1. One language in the project (JavaScript) means it's simply easier to code.
1. There's opportunity to share code between the frontend and backend apps or even make the app isomorphic.
1. Node.js allows server-side rendering. This means you can render your page on the server before it hits the browser, allowing users to see the page quicker. (There are attempts to achieve this with PHP/JS extensions, but for the time being, these do not work with many SPA frameworks like Vue, and if they do, they're much slower).
1. Node has non-blocking I/O and is better at handling concurrent requests (PHP can do this now too, but again, slower).

## Stuck with PHP

Given all of the above, my assumption for why you'd use PHP for a SPA backend is because you must be *stuck with it*, and Laravel is chosen because it's simply the best of a bad situation. 

You might be stuck with PHP if:

<!--more-->

- The core competency of you and your team is PHP and you don't feel comfortable going full JS. 
- You have a legacy code base or infrastructure that is PHP-based and you can't easily change it. 
- Your client is insisting on PHP for whatever reason that they won't budge on ("money" for example...)

All those are actually pretty good reasons to use PHP, albeit not very inspiring ones. And *that's the bit that didn't make sense*... 

How come so many devs passionately choose Laravel when their stack would always be inferior to one with Node.js? Are they just ignorant or too stubborn to acknowledge the glory of full stack JavaScript?

Going back to PHP and working with Laravel for the first time in a few years, I can now see that there was more to the story than I realised.

## Why Laravel is great for a SPA backend

Most developers will mention performance and features when discussing the benefits of a framework, but when performance and features are sufficiently met, the ease of development and maintenance will be what matters most.

Laravel has a mantra of "making developers happy", and a big reason users are so passionate about Laravel is because it really delivers on this. Going to Laravel after a few years with Node.js/Express, I was pretty impressed with how simple and elegant it is.

### Example: syntax

Laravel syntax is expressive and easy for humans to understand. Even if you've never seen Laravel code before, you can probably tell what the following is doing:

```php
<?php

Route::get('api/users/{user}', function (App\User $user) {
  return $user->email;
});
```

But once you break down what it is *actually* doing, there's an even greater level of beauty. You might have already picked up that this is a route that captures incoming GET requests to paths matching `api/users/{user}` where `{user}` is a user ID, but you might *not* have picked up on the following:

1. The argument of the function `$user` type hints the `App\User` class. Laravel's Service Container (explained below) will resolve this and inject an instance of that class in the closure.
2. Laravel knows this is a data model since the `User` class extends the `Eloquent` class (Eloquent is Laravel's ORM). The instance of User you get will be one where the ID matches the corresponding ID from the request URI i.e. `{user}`.
3. If a matching model instance is not found in the database, a 404 HTTP response will automatically be generated.

That's pretty damn elegant.

### Object-oriented frameworks are powerful

JavaScript now has "classes" but it is not naturally an object-oriented (OO) language. PHP is, though, and Laravel makes heavy use of OO design patterns to powerful effect. 

Let's look at one example that I think you'll be impressed with: Laravel's *Service Container*. This is an implementation of an object-oriented design concept known as "inversion of control" that makes dependency injection a breeze. 

Let's say you're creating an app that allows users to crop their images. The images get stored in an Amazon S3 bucket and you'll have a lot of transactions with that bucket throughout your app. You make a helper class called `Bucket` that, when instantiated, can be used like this:

```
$bucket->addFile($someFile);
```

The class you create would look like this:

```php
<?php

namespace App\Helpers;

class Bucket
{
    protected $key;

    public function __construct($key) {
        $this->key = $key;
    }

    protected function authorize() {...}

    public function addFile($file) {...}

    public function deleteFile($file) {...}
}

```

Note that the constructor requires the API key to be passed in, as you obviously don't want to hard code it, so you'll instantiate your class at the top of every file like this:

```php
<?php

$key = config('amazon.api_key');
$bucket = new App\Helpers\Bucket($key);

$bucket->addFile($someFile);
```

The problem is that this same code will need be repeated in *every file*, not only adding repetition, but also the potential for bugs.

The Service Container allows you to do that setup one time, then inject it anywhere. Here's the setup:

```php
<?php

$this->app->bind('App\Helpers\Bucket', function ($app) {
  $key = config('amazon.api_key');
  return new App\Helpers\Bucket($key);
});
```

Now `app` helper can inject a fresh, pre-configured `Bucket` object anywhere:

```php
<?php

$bucket = app('App\Helpers\Bucket');
$bucket->addFile($someFile);
```

The coolest thing is that you don't need to use the `app` helper in functions as you can type hint in the profile and Laravel will automatically resolve it from the Service Container:

```php
<?php

public function someFunction(\App\Helpers\Bucket $bucket) 
{
  // $bucket is a pre-configured `Bucket` object
  $bucket->addFile($someFile);
}
```

## TL;DR

If you want to make a real-time app with a ton of concurrent users, or if server-side rendering is critical, then sure, Node.js is the clear choice. But for the broader question of whether Laravel could contend against Node as a SPA backend, I'd say definitely yes, as Laravel:

- Is a simple and elegant framework that makes development and maintenance a breeze.
- Uses powerful object-oriented design features to help you architect a well-structured backend.

If you look at the last few releases of Laravel (e.g 5.3 adding Vue as the default JS framework, and 5.4 adding Laravel Mix as a Webpack API) it's clear that the creators intend for Laravel to stay relevant in the world of SPAs.

> If you're interested in hearing when my book *Vue.js Full Stack Development* will be done, [jump on my newsletter](http://vuejsdevelopers.com/newsletter) as I'll have more info about it soon!

## Epilogue: Server rendering alternatives

It is a bit of downside to Laravel (and, to be fair, all other non-JS frameworks) that server-side rendering SPAs is often not an option. For example, [Vue.js only supports SSR with Node.js](https://github.com/vuejs/vue/issues/4101#issuecomment-258194824). 

However, one alternative to SSR that is often suitable is *pre-rendering*. With this approach you run your app before deploying it, capture the page output and replace your HTML files with this captured output. It’s pretty much the same concept as SSR except it’s done *pre-deployment* in your development environment, *not a live server*. It has certain caveats, but may be a sufficient solution for your SPA.

I wrote more about pre-rendering with Laravel in a [previous article](/2017/04/01/vue-js-prerendering-node-laravel/).

The other option is to run a Node server parallel to your Laravel server and let Node handle the SSR.

> *Get the latest Vue.js articles, tutorials and cool projects in your inbox with the [Vue.js Developers Newsletter](http://vuejsdevelopers.com/newsletter/?jsdojo_id=cjs_lbn)*
