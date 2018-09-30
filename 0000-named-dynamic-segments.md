- Start Date: 2018-09-27
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Named dynamic segments

## Summary

> One paragraph explanation of the feature.

## Motivation

> Why are we doing this? What use cases does it support? What is the expected
outcome?

The [open "Router link component and routing helpers" RFC](https://github.com/emberjs/rfcs/pull/339) proposes new helpers and a new `Link` component. One important difference of the proposed APIs will be named dynamic segments. While most of that RFC could be implemented as an addon on the existing public API, the essential building block for this is the router service.

While the concept of named dynamic segments maybe could be implemented in front of the router service this will lead to two different and so confusing APIs. So this RFC proposes a change to the existing router service to move to named dynamic segments.
This will also make the [open "Always run model hook"](https://github.com/emberjs/rfcs/pull/283) obsolete.

Also this RFC will change some of the [Router Service RFC](https://github.com/emberjs/rfcs/blob/master/text/0095-router-service.md).

## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

The current `transitionTo`, `isActive` and `replaceWith` implementations on the RouteService have the following signature:

```
transitionTo(routeNameOrUrl, ...models, options)
replaceWith(routeNameOrUrl, ...models, options)
isActive(routeNameOrUrl, ...models, options) 
```

The `options` then object has `queryParams` property.

Note: The `...models` signature already is a bit less-ideal for typescript because it's not possible to write a type definition for the `options` argument.

This RFC proposes to add a `dynamicSegments` property to the `options` hash. `dynamicSegments` is a hash of the named dynamic segments. So for this route definition:

```
this.route('blog-post', { path: '/blog-post/:blog_post_id' }, function() {
  this.route('comment', { path: '/comment/:comment_id' });
})
```

This will be possible ways to call `transitionTo`:

```
transitionTo('blog-post', { dynamicSegments: { blog_post_id: '1' } });
transitionTo('blog-post.comment', { dynamicSegments: { blog_post_id: '1', comment_id: '1' } });

```

`dynamicSegments` will *not* take model instances but only primitive datatypes. This means the `model` hook on the route will always be called.
This will make the [open "Always run model hook"](https://github.com/emberjs/rfcs/pull/283) obsolete.
This also means the `serialize` function on the route will be obsolete.


### Transition to the new API

First the new API needs to work side-by-side with existing API.
Next the old API on the router service and the `serialize` function on the Route will be deprecated.
Next on a major release the old API will be removed.


## How we teach this

Essentially this will not increase the API surface. Instead it will actually reduce the API surface.
We need to update the guides and the API docs to use the new API. 

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

The possibility to pass model instances to the transition functions will be removed. In the end calls to `transitionTo` will be longer. Also this removes some caching possibilities that could be considered *nice*. However the overall benefit from the cleaner APIs seems worth this.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

We could keep serialize and implement some api like `dynamicSegments: { blogPost: myBlogPost }` and let `serialize` transform this to `{ blog_post_id: myBlogPost }`. However `serialize` feels very magically and not in the spirit of the current attempts to make ember easier to learn.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?

Do we want `dynamicSegments` side by side with `queryParams`? We could also do `transitionTo({ blog_post_id: "1" }, { queryParams: {...} })`, however this makes it problematic to implement this with the old API side-by-side.