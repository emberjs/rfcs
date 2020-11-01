---
Start Date: 2019-08-05
Relevant Team(s): Ember.js, Learning
RFC PR: https://github.com/emberjs/rfcs/pull/523
Tracking: (leave this empty)

---

# Provide `@model` named argument to route templates

## Summary

Allow route templates to access the route's model with `@model` in addition to
`this.model`.

Before:

```hbs
{{!-- The model for this route is the current user --}}

<div>
  Hi <img src="{{this.model.profileImage}}" alt="{{this.model.name}}'s profile picture"> {{this.model.name}},
  this is a valid Ember template!
</div>

{{#if this.model.isAdmin}}
  <div>Remember, with great power comes great responsibility!</div>
{{/if}}
```

After:

```hbs
{{!-- The model for this route is the current user --}}

<div>
  Hi <img src="{{@model.profileImage}}" alt="{{@model.name}}'s profile picture"> {{@model.name}},
  this is a valid Ember template!
</div>

{{#if @model.isAdmin}}
  <div>Remember, with great power comes great responsibility!</div>
{{/if}}
```

## Motivation

With the Octane release, templates are taking on a more important role in
idomatic Ember apps. As templates become more self-sufficient, many cases where
a JavaScript class was needed in the past (e.g. to customize the "wrapper"
element) can now be expressed with templates alone. This is a direction we will
continue post-Octane.

We would like to update the learning materials (guides) to focus on teaching
the template-centric component model. For example, components can be introduced
as a way to break up large templates into smaller, named pieces, similar to
refactoring a big function into smaller ones. Then, we can layer on making them
reusable through passing arguments. Next, we can introduce the idea of passing
a block and yielding. Finally, we can introduce the component class when we are
ready to discuss adding behavior to the component.

As you can see, we can accomplish quite a lot with template-only components in
Octane, and focusing on teaching templates first-and-foremost would provide
a gentle learning curve for developers and designers who are comfortable with
HTML but perhaps new to Ember (or even JavaScript). With this flow, the concept
of a component class, and consequently, the idea of a "`this` context" in a
template comes up quite late.

This presents a problem, as we may want or need to teach route templates before
that was introduced. As the model can only be accessed through `this.model` in
a route template, we would be forced to introduce that concept (and the contept
of a controller) at an earlier time than would be ideal.

Providing `@model` to route templates would solve this problem quite elegantly.
We will now be able to introduce the concept of arguments (`@name`) once and it
can be applied consistently between component and route templates.

This also aligns with the general mental model that arguments are things that
are passed into the template from the outside (which is true in the case of the
route model).

This can also be thought of as a small incremental step in the bigger picture
of reforming route templates and removing controllers from Ember. Specifically,
it moves us a bit closer to the mental model that controllers/route templates
are just a "special" kind of component. We expect to continually unify them and
remove the remaining differences, and this is a step towards that direction.

## Detailed design

Internally, route templates are _already_ modelled as components at the Glimmer
VM layer. To implement this, we would "synthesize" a named argument `@model`
containing the resolved route model, i.e. the same value as `this.model` on the
controller instance.

Just like the "reflected" named arguments in classic components, mutating
`this.model` on the controller instance would _not_ change the value of
`@model`. In practice, this seems unlikely to be relied upon and probably
considered a bad practice (it does not change the URL, does not affect what is
returned by `route.modelFor`, etc). In any case, this is consistent with the
general behavior for named arguments, in that they are immutable and should
always reflect what was "passed in" from the caller.

## How we teach this

In the guides, we should teach that `@names` are for things that are "passed
in", i.e. arguments to the template.

In the tutorial, basic components concepts (template-only components, invoking
a component with args, using the args with the template, etc) should be taught
before the `model` hook is encountered. At that point, explaining that `@model`
is passed into the component from the route, based on the resolution of the
async `model` hook, will be quite natural.

For reference, here is a relevant section from the [work-in-progress Octane
tutorial](https://github.com/ember-learn/guides-source/pull/1002). For context,
up until this point, we have taught all the basic component concepts above, plus
adding a (Glimmer) component class, adding instance variables and getters to the
component, accessing those with `{{this.*}}` in the template, etc.

> So far, we've been hard-coding everything into our `<Rental>` component. But
> that's probably not very sustainable, since eventually, we want our data to
> come from a server instead. Let's go ahead and move some of those hard-coded
> values out of the component in preparation for that.
>
> We want to start working towards a place where we can eventually fetch data
> from the server, and then render the requested data as dynamic content from
> the templates. In order to do that, we will need a place where we can write
> the code for fetching data and loading them into the routes.
>
> In Ember, [*route files*](todo://) are the place to do that. We haven't
> needed them yet, because all our routes are essentially just rendering static
> pages up until this point, but we are about to change that.
>
> Let's start by creating a route file for the index route. We will create a
> new file at `app/routes/index.js` with the following content:
>
> ```js {data-filename="app/routes/index.js"}
> import Route from '@ember/routing/route';
>
> export default class IndexRoute extends Route {
>   async model() {
>     return {
>       title: 'Grand Old Mansion',
>       owner: 'Veruca Salt',
>       city: 'San Francisco',
>       location: {
>         lat: 37.7749,
>         lng: -122.4194,
>       },
>       category: 'Estate',
>       type: 'Standalone',
>       bedrooms: 15,
>       image: 'https://upload.wikimedia.org/wikipedia/commons/c/cb/Crane_estate_(5).jpg',
>       description: 'This grand old mansion sits on over 100 acres of rolling hills and dense redwood forests.',
>     };
>   }
> }
> ```
>
> There's a lot happening here that we haven't seen before, so let's walk
> through this. First, we're importing the [*`Route` class*](todo://) into the
> file. This class is used as a starting point for adding functionality to a
> route, such as loading data.
>
> ```js
> import Route from '@ember/routing/route';
> ```
>
> Then, since we are extending the `Route` class into our _own_ `IndexRoute`,
> which we are also exporting so that the rest of the application can use it.
>
> ```js
> export default class IndexRoute extends Route {
>   // ...snip...
> }
> ```
>
> So far, so good. But what's happening inside of this route class? We
> implemented a [*async method*](todo://) called `model()`. This method is also
> known as the [*model hook*](todo://).
>
> The model hook is responsible for fetching and preparing any data that you
> need for your route. Ember will automatically call this hook for when
> entering a route, so that you can have an opportunity to do what you need to
> get the data you need. The object returned from this hook is known as the
> _model_ for the route (surprise!).
>
> Usually, this is where we'd fetch data from a server. Since fetching data is
> usually an asynchronous operation, the model hook is marked as `async`. This
> gives us the option of using the `await` keyword to wait for the data
> fetching operations to finish.
>
> We'll get to that bit later on. At the moment, we are just returning the same
> hard-coding model data, extracted from the `<Rental>` component, but in a
> JavaScript object (also known as [_POJO_](todo://)) format.
>
> So, now that we've prepared some model data for our route, let's use it in
> our template. In route templates, we can access the model for the route as
> `this.model`. In our case, that would contain the POJO returned from our
> model hook.
>
> To test that this is working, let's modify our template and try to render
> the `title` property from our model:
>
> ```handlebars {data-filename="app/templates/index.hbs"}
> <h1>{{this.model.title}}</h1>
>
> <div class="rentals">
>   <ul class="results">
>     <li><Rental /></li>
>     <li><Rental /></li>
>     <li><Rental /></li>
>   </ul>
> </div>
> ```
>
> If we look at our page in the browser, we should see our model data reflected
> as a new header.
>
> <!-- TODO: screenshot -->
>
> Awesome! Looking good.
>
> <!-- TODO: Update the below if https://github.com/emberjs/rfcs/pull/523 is merged. -->
>
> > Zoey says...
> >
> > The `this` in `this.model` does _not_ refer to the route object. You
> > _cannot_ add instance variables or getters on the route class and have
> > access to them here. It's a good idea to keep your route template simple
> > and minimal &mdash; if you need to add state or getters, just add a
> > component and call it from your route template!
>
> Ok, now that we know that we have a model to use at our disposal, let's
> remove some of the hard-coding that we did earlier! Instead of explicitly
> hard-coding the rental information into our `<Rental>` component, we can pass
> the model object to our component instead.
>
> Let's try it out. First, let's pass in our model to our `<Rental>` component
> as the `@rental` argument. We will also remove the extraneous `<h1>` tag we
> added earlier, no that we know things are working:
>
> ```handlebars {data-filename="app/templates/index.hbs"}
> <div class="rentals">
>   <ul class="results">
>     <li><Rental @rental={{this.model}} /></li>
>     <li><Rental @rental={{this.model}} /></li>
>     <li><Rental @rental={{this.model}} /></li>
>   </ul>
> </div>
> ```
>
> ...snip...
>

As you can see, there are some awkwardness explaining `this.model` at this
point. Knowing that there is a `this` context on the template, it is only
natural to inquire what _is_ the `this` we are referring to here.

Up until this point in the tutorial, the only place where we have access to a
`this` object in the template is a component with a class. Since the route
class is the only related class we made at this point, and it happens to have
a `model` property (a method) on it, one natural conclusion is to think that
`this` refers to the route instance, and `this.model` refers to the model hook.

> Note: you don't always have a `this` context in templates. Template-only
> components _do not_ have a `this` context. Since template-only components
> make up roughly half of the components in the tutorials so far, _having_ a
> `this` from the template is a notable thing that stands out.

But that completely the wrong mental model! To avoid that, we put in a "Zoey
says..." sidebar that explicitly denies that incorrect mental model, stating
what it _is not_, without really explaining what it _is_.

To fully explain what is going on, we would have to pause and take a detour to
explain everyone's least favourite part of the Ember programming model –
controllers. This would be extremely disruptive to the teaching flow, to say
the least, but also serves no purpose at all.

We opted to avoid going down that rabit hole and just treated it as a
boilerplate syntax you have to type.

If we are able to explain this using `@model` instead, it would make things a
lot smoother here.

We wouldn't have to go into the details of _how_ things get invoked internally.
At the end of the day, it is still just a piece of syntax you type to get the
job done, but at least it is consistent with the general mental model that
`@names` means that thing was passed in, and it does not trigger the questions
about what `this` is.

It is also _not_ inconsistent with the fact that there is, in fact, a `this`
context on the template. The usage and preference of using `@names` does not
preclude the existance of a class (which provides the `this` context), as in
the case of `@ember/components`.

As for the concrete changes that need to be made in the documentation:

* We will explain that `@model` is passed into the component from the route.
  For example, in the `Route` class' `model` hook documentation, we can say
  something like:

  > The model hook is responsible for fetching and preparing any data that you
  > need for your route. Ember will automatically call this hook for when
  > entering a route, so that you can have an opportunity to do what you need to
  > get the data you need. The object returned from this hook is known as the
  > _model_ for the route.
  >
  > Note that since this is an `async` hook, if a promise is returned, it will
  > be automatically `await`ed by Ember, and the _resolved value_ of the
  > promise, as opposed to the promise object itself, will become the model of
  > the route.
  >
  > The model object can be accessed from the route's template as using the
  > `@model` argument. By default, the controller can also accessed the route's
  > model from `this.model`. The latter behavior is customizable, see the
  > _setupController_ method for details.
  >
  > The model of a route should be treated as read-only. For example, the
  > controller _should not_ mutate its `this.model` property, as doing so will
  > cause it to get out of sync with the rest of the system. Specifically,
  > doing so will _not_ update the current URL, the `@model` argument in the
  > template, the router service, etc. Instead, a _route transition_ should be
  > performed.

* Guides and API docs should be updated to prefer `@model` in route templates.
* API docs will still document the `model` property.
* We will remove any examples that uses `{{this.model}}` in templates, or replace them with `{{@model}}`. i.e. we won’t be documenting using `this.model` in templates, even thought it would “work”.

## Drawbacks

In some applications, developers have developed a pattern to override the
`setupController` method to assign the model to a different property on the
controller other than the default `model` naming convention.

Since this RFC does not provide any way to customize the name of the argument,
developers using this pattern will have to choose between one of the following
two options:

1. Refactor/revert to the default behavior of `setupController` and use the
   default `model` property. This makes it cognitively easy to understand the
   `@model` usages in the route templates.

2. Stick with the alternative names and avoid using `@model` in the route
   templates, until there is a way to customize both names at the same time.

Technically, it is also possible to use one name in the controller and
`@model` in the template. This doesn't cause any issues for the system, but
maybe confusing for the developers.

In the future, we expect to generalize the `model` hook and `@model` into
allowing passing arbitrary arguments into the template, perhaps replacing
the `model` hook with something like an `args` hook that returns a POJO of
arguments to pass.

## Alternatives

We can do nothing.

## Unresolved questions

None?
