- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Migration away from controllers

## Summary

_The goal is for components to make up the entirety of the view layer_.

 - Requires:
   - [RFC 496: Handlebars Strict Mode](https://github.com/emberjs/rfcs/pull/496)
   - [RFC 454: SFC & Template Import Primitives](https://github.com/emberjs/rfcs/pull/454)
   - [RFC 380: Add queryParams to the router service](https://github.com/emberjs/rfcs/pull/380)

 - Provide alternative APIs that encompass existing behavior currently implemented by controllers.
   1. query params
   2. passing state and handling actions to/from the route, loading, and error templates

 - Out of scope:
   - Deprecating and eliminating controllers -- but the goal of this RFC is to provide a _path_ to remove controllers from Ember altogether. 

 - Not Proposing
   - Adding the ability to invoke actions on the route.
   - Any changes to the `Route` or `Controller` as they exist today
   
This'll explore how controllers are used and how viable alternatives can be for migrating away from controllers so that they may eventually be removed.

## Motivation

 - Many were excited about [Routable Components](https://github.com/ef4/rfcs/blob/routeable-components/active/0000-routeable-components.md) (This RFC is _not_ "Routable Components").
 - Many agree that controllers are awkward, and should be considered for removal (as mentioned in many #EmberJS2019 Roadmap blogposts).
 - There is, without a doubt, some confusion with respect to controllers:
   - should actions live on controllers?
   - how are they different from components?
   - controllers may feel like a route-blessed component (hence the prior excitement over the Routable-Components RFC)
   - controllers aren't created by default during `ember g route my-route`, so there is a learning curve around the fact that route templates can't access things on the route, and must create another new file, called a controller.
 - For handling actions from the UI, component-local actions and actions that live on services are a better long-term maintenance pattern. Dependency injection especially from the store and router services for fetching / updating data as well as contextually redirecting to different routes. 
 - State and Actions on the controller encourage prop drilling where arguments are passed through many layers of components which make maintenance more difficult.


## Detailed design

First, what would motivate someone to use a Controller today? Borrowing from [this blog post](https://medium.com/ember-ish/what-are-controllers-used-for-in-ember-js-cba712bf3b3e) by [Jen Weber](https://twitter.com/jwwweber): 

1. You need to use query parameters in the URL that the user sees (such as filters that you use for display or that you need to send with back end requests)  

2. There are actions or variables that you want to share with child components  

3. (2a) You have a computed property that depends on the results of the model hook, such as filtering a list

All of this behavior can be implemented elsewhere, and most of what controllers _can_ do is already more naturally covered by components.

### 1. Query Params

See this RFC for [adding query params to the router service](https://github.com/emberjs/rfcs/pull/380), query-param management would be pulled out of the controller entirely.

### 2. Passing State and Handling Actions to/from the Template

In order to fully eliminate the controller, there _must_ be a way to handle computed properties or state along with responding to actions from the template. Components can cover this already, and would be an intuitive way to interact with the view and behavioral layer.

Currently, controllers + templates must be used in the app by having:
  - `app/controllers/{my-route-name/or-path}.js`
  - `app/templates/{my-route-name/or-path}.hbs`

The new behavior would allow components to be imported from anywhere in the app, and assigned to static properties on the Route.

```ts
import FooComponent, { LoadingComponent, ErrorComponent } from './components/Foo';
import Route from '@ember/routing/route';

export default class extends Route {
  static render = FooComponent;
  static loading = LoadingComponent;
  static error = ErrorComponent;

  async model() {
    const contacts = await this.store.findAll('contact');
    
    return { contacts };
  }
}
```

Everything is encapsulated in components so that intent is clear as to which part of the template is responsible for what behavior.
Additionally, all three properties may be set to the same component if it was desired to manage loading, error, and success states within components.

This RFC proposes _three_ new static properties on the Route class.

#### **render**

This is the component that will be rendered after the model hook resolves.
Today, there is a default template generated per-route that only contains `{{outlet}}`. In order to not break that behavior, the `render` property must not be required until both controllers and the route template are removed from the framework, after a deprecation RFC and the deprecation process).

If set, and there exists a classic route template, an error will be thrown saying that there is a conflict.

The render component will receive the `@model` argument, which represents the value of a resolved `model` hook.

#### **loading**

This is the component that will be rendered before the model hook has resolved, if present. 

If set, and there exists a classic loading template, an error will be thrown saying that there is a conflict.

The loading component does not receive any arguments.

#### **error**

This is the component that will be rendered if an error is thrown as they are thrown today.

If set, and there exists a classic error template, an error will be thrown saying that there is a conflict.

The error component will receive the `@error` argument, which will be the error object that is presently passed to the error action.

## How we teach this

TODO: write this

## Drawbacks

- Routes may feel like they are configuration (this may be fine)

## Unresolved questions

 - Should this RFC include the idea that's been floating around recently where `Route` defines an `arguments` getter where those arguments are splatted onto the `render` component, rather than receiving a single `@model` argument. This would mean that the `Route` would need to have a way to manage the resolved model so that `arguments` doesn't need to worry about pending promises.
 - For error templates today, is `this.model` the error?
