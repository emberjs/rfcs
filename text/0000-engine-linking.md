- Start Date: 2016-02-17
- RFC PR: [emberjs/rfcs#122](https://github.com/emberjs/rfcs/pull/122)
- Ember Issue: (leave this empty)

# Summary

This RFC proposes a method of linking to routes external to an Engine. The main idea presented is introducing a new Engine configuration option, `externalRoutes`, and an associated helper, `{{link-to-external}}`, as the methodology for linking outside of an Engine. A quick example of linking to an external `blog` would look like so:

```js
// some-engine/engine.js
import Engine from 'ember-engines/engine';
export default Engine.extend({
  // ...
  dependencies: {
    externalRoutes: [
      'blog'
    ]
  }
});
```

Then, in your template, you do:

```hbs
{{! some-engine/application.hbs !}}
{{#link-to-external "blog"}}Blog{{/link-to-external}}
```

# Motivation

Engines bring with them the tremendous benefits of encapsulation and code isolation, but there are use cases where it is desirable to break those boundaries by linking into an external context, such as a consuming Application or sibling Engine.

One prominent use case is for "tabbed applications" that might consist of multiple areas which can be represented by Engines. Those areas contain separate functionality but will often link into each other for various user flows. This is a pattern frequently seen in ambitious web applications (Facebook, LinkedIn, Twitter, etc.) that could gain significant benefits from the performance optimizations promised by Engines (especially async Engines).

There are also simple use cases that can benefit from being able to link between contexts. One common example would be linking to the "home page" of an application. Even if your Engine is totally unaware of its parent, there are still times where it would be reasonable to direct them to the "home" route of their parent context.

# Terminology

- **Context**: A context, as used in this RFC, either refers to an Engine or an Application. These constructs are isolated in nature and so we should think of them as being different contextual areas with regards to linking. While linking between these contexts will always require an Engine (since there can't be more than one Application), it could very well be that you're linking from an Engine context to an Application context and so we'll generically use "context" as way to describe both.

# Detailed Design

In designing a solution to this problem, we wanted to make sure it:

- preserves isolation by default,
- does not dramatically affect the learning curve,
- is flexible enough to account for a multitude of use cases, and
- allows consumers to determine the destination of the links

These constraints and initial feedback led to the development of using `{{link-to-external}}` with an `externalRoutes` configuration.

## Isolated By Default

Developing in isolated contexts means we can't assume knowledge of the global context in which our code is mounted. This means that we might not be able to use an absolute path for constructing a `{{link-to}}` or other linking mechanism. Instead, links should be constructed relative to their context's boundary (e.g., the mounting point for a routable engine) by default.

This is the default behavior for links in engines currently and makes sense given their isolated nature. In practice this looks like:

```hbs
// in route "foo.bar.baz", where "bar" is
// the mount point of an engine
{{#link-to "boo"}}Boo!{{/link-to}} => goes to "foo.bar.boo"
```

## Linking To An External Context

Assuming the above default behavior for links, we can look at the solution for linking between different contexts (or, links that break isolation). In order to support those use cases, we propose introducing the concept of `externalRoutes` to Engines.

### Declaring and Defining External Routes

The `externalRoutes` for an Engine will be declared in the Engine's dependencies and later defined by the consumer of the Engine. Here's a simple example:

```js
// some-engine/engine.js
import Engine from 'ember-engines/engine';
export default Engine.extend({
  // ...
  dependencies: {
    externalRoutes: [
      'blog'
    ]
  }
});
```

```js
// some-app/app.js
import Ember from 'ember';
export default Ember.Application.extend({
  engines: {
    someEngine: {
      dependencies: {
        externalRoutes: {
          blog: 'path.to.blog'
        }
      }
    }
  }
});
```

This API gives us great flexibility in allowing the consumers to define where an Engine should be linking to and allows easy targeting of both ancestor and sibling routes. By allowing the consumer to define the meaning of routes external to the Engine, we decouple the various contexts from each other which makes reuse of Engines in different applications easier, but at the cost of requiring a small amount of additional setup.

### Using External Routes

Using the external routes will take advantage of new "external routing" APIs, such as `{{link-to-external}}` and `transitionToExternal`. These will be equivalent to the non-external APIs in every way except that the valid route names to use will be those from the `externalRoutes` declaration. This means that if we build on the prior example, we could do the following in our templates:

```hbs
{{#link-to-external "blog"}}Blog{{/link-to-external}}
```

And this in our JavaScript:

```js
goToBlog() {
  this.transitionToExternal("blog");
}
```

The value passed to these "external" APIs will be used to look up the corresponding route value from the `externalRoutes` definitions.

By keeping the concept of external linking separate from internal (or, normal) linking, we help keep the learning curve down by not muddying the two concepts. Instead, developers will learn how normal linking works and then as they begin to work with Engines they can learn about the concepts of external linking. In fact, these APIs should likely only be avaialable within the context of an Engine.

#### An Alternative To New APIs

One alternative approach here is to introduce an `external-route` helper that could be used with existing routing APIs, something like:

```hbs
{{#link-to (external-route "blog")}}Blog{{/link-to}}
```

This would allow us to avoid needing to introduce multiple new routing APIs. However,  since `{{link-to}}` will be scoped to the Engine by default, we will likely need to introduce some way of differentiating between external links and internal links. Two options in this regard are to use special characters in the string, which is brittle, or represent routes as non-strings. There are several drawbacks to both of these approaches:

- any existing routing APIs will still need to change internally (to account for external vs. internal routes),
- it potentially adds an element of polymorphism into those APIs,
- the way to define routes has now diverged into two different forms, and
- it is somewhat magical which risks addons having to work with private APIs to mimic the behavior

For those reasons, and keeping a clear separation of concerns, we think it makes more sense to simply introduce new, separate interfaces.

### Nested Engine Edge Case

When defining an external route for a nested Engine, it is conceivable to want to pass forward a previously defined external route. In order to support this, we would likely want to introduce a helper to lookup the path of external routes:

```js
// some-engine/engine.js
import Engine from 'ember-engines/engine';
import { getExternalRoute } from 'ember-engines/routes';
export default Engine.extend({
  dependencies: {
    externalRoutes: [
      'adminPanel'
    ]
  },

  engines: {
    someNestedEngine: {
      dependencies: {
        externalRoutes: {
          adminPanel: getExternalRoute('adminPanel')
        }
      }
    }
  }
});
```

### External Link Errors

External routes will be expected to be defined at the time of Engine instantiation, similar to the behavior of Service dependencies. If a route is not defined, an error will be thrown similar to:

```bash
ERROR: Expected an external route 'blog' to be defined for Engine 'editorial'.
```

_Exact wording is up for debate._

If the external route is defined, but it is invalid, then a run-time error will occur when the application attempts to reference it. This should be similar to current runtime errors for invalid routes.

## Implementation and Other Considerations

Implementation of these new features should be relatively straightforward. When an external route is being used through one of the new APIs, we will simply look it up from the `externalRoutes` definitions and then defer to the original implementations of the methods without any scoping semantics.

The lookup function for `externalRoutes` will likely be the internals of the `getExternalRoute` helper function proposed in the "Edge Cases" section.

### Asynchronous Considerations

One primary concern is being able to link to Engines that might not have been loaded yet. This design, which can leverage much of the existing `{{link-to}}` functionality, means that those async issues (such as handling query params, loading/transition states, and knowing available, unloaded routes) should mostly "just work" by virtue of supporting `{{link-to}}`.

Since knowing available routes and transition information is an issue of the router, those problems can be solved at that level without changing this API. Additionally, [default query params](https://github.com/emberjs/rfcs/pull/113) and [dynamic segments](https://github.com/emberjs/rfcs/pull/120) both have relevant RFCs for handling their usage before the entire Engine has loaded.

If, however, there is some need for this API to change to account for async Engines, it will also mean that it needs to be solved for all usages of `{{link-to}}` and is not a specific concern of linking between contexts.

### Testing

Testing an Engine with external links should be relatively straightforward with this design. Since the configuration allows the consuming app to customize the external routing behavior, it would be easy to stub those out with simple routes in your dummy application for any acceptance tests. Additionally, there should be no issues with unit and integration tests as those are already setup to not resolve routes since there is no router present.

# How We Teach This

This RFC would introduce several new interfaces, each of which will require an addition to the Ember guides to discuss its behavior. However, since these are specific concerns of Engines, they can be introduced with those relevant guides. In fact, as mentioned above, it would make sense to limit the availability of these interfaces to the boundaries of Engines.

This new syntax is not critical to new users, only those that desire to use Engines, and since Engines themselves are new, we don't anticipate any major hurdles with educating existing users (as they will also need to reference the guides).

# Drawbacks

## Expanded API Area

The biggest obvious drawback here is the number of new API surface area introduced. This is obviously undesirable, but as mentioned the alternatives have some substantial drawbacks and would still require new API.

Since these interfaces won't be needed by all users and are relatively straightforward to use, we don't anticipate this dramatically affecting the learning curve or usability of Ember.

## Favors Configuration

Ember's motto has long been "convention over configuration", but in this case the amount of flexibility given by opting for configuration seems to be the most viable solution to this problem. The configuration approach allows for easier testing and greater consumer control than other convention based approaches.

# Alternatives

Several other approaches were considered for linking between different contexts.

## Parent-Traversal Syntax

The first iteration of this RFC proposed using a parent-traversal micro-syntax in the route DSL. Specifically, it advocated using `^` as a way to denote linking relative to a parent context. This approach had many of the benefits that the current approach has, but had several major drawbacks.

First, introducing the `^` to routes would mean that we now need to know information about the context tree (e.g., the tree formed by the Engines in an Application) and the route tree. This adds both conceptual and runtime complexity.

Second, and more importantly, using a parent-traversal approach binds your links to the exact structure of the route tree. While this can be advantagous in some ways, it makes reuse harder. Instead, favoring configuration allows you to decouple from the tree while still being able to link anywhere in the route tree.

Finally, expanding the micro-syntax used by routes introduces one more thing for developers to learn, though it is comparatively less than learning an entirely new construct for linking. This would actually be the one area this approach is more desirable than the one given above.

## Other Types of Relative Paths

Using another syntax for relative pathing that would be more familiar to users (as opposed to `^`) was also a consideration. In particular, we considered using `../` or filesystem style traversals, since this would require little introduction for developers. This had two major issues though (in addition to the ones outlined for `^`):

**First**, this syntax implies _route-relative_ traversal as opposed to _context-relative_ traversal. This would eliminate the "dual-tree drawback" mentioned above, but means that developers would more than likely need to introduce hacks to work around the notion of "one route up".

This problem arises because "one route up" is different depending on where you are in the context's routing structure. For example, if you have a component that desired to link to it's parent's `foo` route you might use: `{{link-to "../foo"}}`. This is fine, until you attempt to render your component at two different depths in the route tree, such as `boo.bar` and `baz`, the first will link to `boo.foo` and the second will link to `foo`. This is not ideal and likely to occur frequently for reusable components.

On the other hand, one context up is always consistent within a given context. The only time this would become an issue is if you have a component being shared across multiple contexts that are at varying depths in the context tree which also require inter-context linking – a very small use case. To address this we investigated adding a "depth" operator, `(^)`, which resolves to `^ * number of contexts deep` but we feel the limited use cases do not warrant it being a core feature at this time.

**Second**, the `../` syntax, while familiar, is inconsistent with the current route syntax. Routes are currently defined with `.` as the separator, this new syntax would also introduce `/` as a separator which increases complexity and looks bad.

Nonetheless, since context-relative pathing and route-relative pathing are relatively independent constructs, we are not closing off the option of expanding this way in the future.

## `link-to-parent` or Similar Construct

Another approach explored was to introduce a new construct called `link-to-parent` or something similar. This would be functionally equivalent to what we've established as `^` without the need to expand the route path syntax at all.

However, this approach introduces problems with flexibility. In particular, there is no obvious solution for how to traverse multiple contexts. `link-to-parent` could accept a secondary argument to specify which context to route relative to, but this is now splitting the route information into two distinct pieces which will make code more verbose and conceptually more difficult.

Another approach to solve flexibility was to allow nested usage of the helper, something akin to `{{link-to-parent (link-to-parent "foo")}}` but this is obviously too verbose and would quickly become unwieldy.

# Unresolved Questions

- Does this introduce too much new surface area? Do the benefits of using `external-route` or a similar helper outweigh the drawbacks?
