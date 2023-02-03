---
stage: accepted
start-date: 2021-11-14
release-date: 
release-versions:
teams: # delete teams that aren't relevant
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/774 
---

# Deprecate Implicit Record Loading in Routes

## Summary

This RFC seeks to deprecate and remove the default record loading behaviour on Ember's Route. By consequence, this will also deprecate and remove the default store on every Route.  This behaviour is likely something you either know you have or do not know but it may be occuring in your app.

```js
export default class PostRoute extends Route {
  beforeModel() {
    // do stuff
  }

  afterModel() {
    // do stuff
  }
}
```

In this example, the `model` hook is not defined.  However, Ember will attempt to try a few things before rendering this route's template.

1. If there is a `store` property on your route, it will attempt to call it's `find` method.  Assuming you have `ember-data` installed, you may be expecting this. The arguments will be extracted from the params.
  a. For example, if a dynamic segment is `:post_id`, there exists logic to split on the underscore and find a record of type `post`.  

2. As a fallback, it will attempt to define a `find` method and use your `Model` instance's find method to fetch.  If a `Model` cannot be found or if the found `Model` does not have a `find` method, an assertion is thrown.

## Motivation

If a user does not define a `model` hook, a side effect of going to the network has long confused developers.  Users have come to associate `Route.model()` with a hook that returns `ember-data` models in the absense of explicit injection. While this can be true, it is not wholly true. New patterns of data loading are becoming accepted in the community including opting to fetch data in a Component or using different libraries.

By removing this behaviour, we will encourage developers to explicitly define what data is being fetched and from where.

## Detailed design

We will issue a deprecation to [`findModel`](https://github.com/emberjs/ember.js/blob/017b11e2f58880869a5b8c647bf7f3199fc07f26/packages/%40ember/-internals/routing/lib/system/route.ts#L1376) notifying the user that if they want to continue this behaviour of attempting to fetch a resource implicitly, they should try and replicate with their own explicitly defined `model` hook. 

In addition, we will include an [optional feature](https://github.com/emberjs/ember-optional-features) to disable this feature and clear the deprecation.

In v6.0.0, we will remove `findModel` and logic to determine arguments for this method.  This will not remove returning the `transition` context when no `model` hook is defined.

## How we teach this

Most of this behaviour is lightly documented.  Developers often come to this by mistake after some difficult searching. First, we will have to remove this sentence from the [docs](https://guides.emberjs.com/release/routing/defining-your-routes/#toc_dynamic-segments).

> The first reason is that Routes know how to fetch the right model by default, if you follow the convention.

A direct one to one replacement might look like this.

```js
import { inject as service } from '@ember/service';

export default class PostRoute extends Route {
  // assuming you have ember-data installed
  @service store;

  beforeModel() {
    // do stuff
  }

  model({ post_id }) {
    return this.store.findRecord('post', post_id);
  }

  afterModel() {
    // do stuff
  }
}
```

## Alternatives

- Continue to provide fallback fetching behaviour but ensure no `assert` is called for users that have neither a store nor a `Model` with a `find` method.

## Open Questions

## Related links and RFCs
- [Deprecate defaultStore located at `Route.store`](https://github.com/emberjs/rfcs/issues/377)
- [Pre-RFC: Deprecate implicit injections (owner.inject)](https://github.com/emberjs/rfcs/issues/508)
- [Deprecate implicit record loading in routes](https://github.com/emberjs/rfcs/issues/557)
