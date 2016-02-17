- Start Date: 2016-02-17
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

This RFC proposes a method of linking between different contexts, which at this point in time means either an Application or Engine. The main idea presented is introducing `^` (carat) as a symbol to denote traversing contexts in the route DSL. This means that to go from an Engine to its consuming Application you would do something like:

```hbs
{{#link-to "^"}}Home{{/link-to}}
```

Or, programmatically, you could do:

```js
goHome() {
  this.transitionTo('^');
}
```

# Motivation

Engines bring with them the tremendous benefits of encapsulation and code isolation, but there are use cases where it is desirable to break those boundaries by linking into an external context, such as a consuming application or sibling engine.

One prominent use case is by "tabbed applications" that might consist of multiple areas (which can be represented by Engines). Those areas contain separate functionality but will often link into each other for various user flows. This is a pattern frequently seen in ambitious web applications (Facebook, LinkedIn, Twitter, etc.) that could gain significant benefits from the performance optimizations promised by Engines.

There are also simple use cases that can benefit from being able to link between contexts. One common example would be linking to the "home page" of an application. Even if your Engine is totally unaware of it's parent, there are still times where it would be reasonable to direct them to the "home" route of that parent context.

# Terminology

- **Context**: A context, as used in this document, either refers to an Engine or an Application. These constructs are isolated in nature and so we should think of them as being different contextual areas with regards to linking. While linking between these contexts will always require an Engine (since there can't be more than one Application), it could very well be that you're linking from an Engine context to an Application context and so we'll generically use "context" as way to describe both.

# Detailed Design

In designing a solution to this problem, we wanted to make sure it:

- preserves isolation by default,
- does not dramatically affect the API or learning curve, and
- is flexible enough to account for a multitude of use cases.

This led to the development of using `^` as a way to denote context traversal in the route DSL.

_Note: the following examples focus on `{{link-to}}` but they should be easily applied to other routing APIs, such as `transitionTo` or `replaceWith`._

## Isolated By Default

Developing in isolated contexts means we can't assume knowledge of the global context in which our code is mounted. This means that we might not be able to use an absolute path for constructing a `{{link-to}}` or other linking mechanism. Instead, links should be constructed relative to their context's boundary (e.g., the mounting point for a routable engine) by default.

This is the default behavior for links in engines currently and makes sense given their isolated nature. In practice this might look like:

```hbs
// in route "foo.bar.baz", where "bar" is
// the mount point of an engine
{{#link-to "boo"}}Boo!{{/link-to}} => goes to "foo.bar.boo"
```

## Linking Between Contexts

Assuming the above default behavior for links, we can look at the solution for linking between different contexts (or, links that break isolation). In order to support those use cases, we propose introducing `^` to the route syntax to express routing to a _parent context_.

This syntax would represent the top-level of the current context's parent and start routing from there. Note that this means using `^` takes you to the top of the parent context, _which is not necessarily the parent route_. In practice, this would work like so:

```hbs
// Assuming we're in route "foo.bar.baz", where "bar" is an
// engine. This means the default behavior for links will be:
{{link-to "boo"}}           => foo.bar.boo

// If we specify a context switch, the result is relative to
// the parent context's root, not the direct parent route:
{{link-to "^.boo"}}         => boo

// We can then use "^" to route anywhere relative to the parent
// context:
{{link-to "^.foo.fiz"}}     => foo.fiz

// Even back into the current engine:
{{link-to "^.foo.bar.baz"}} => foo.bar.baz
```

Expanding the route syntax in this way accounts for our first two goals: it preserves isolation by default (by not changing any existing behavior) and it does not dramatically affect any API that users must learn (though it adds one more character to the syntax).

Let's look at the flexibility this approach also gives us.

### Specific Linking Use Cases

#### To Descendants

Linking into a child route will work as it always has, though it will be relative to it's current context (Engine or Application). This technically introduces no change, but users will need to adjust their mental model as they learn Engines since there was previously only ever one context.

##### Example:

```hbs
// in route "foo", which is an engine
{{link-to "boo"}}     => foo.boo
{{link-to "bar.baz"}} => foo.bar.baz
```

#### To Ancestors

Linking to an ancestor will require use of the new `^` syntax. This will be required for any route that has a "pivot" point outside of the Engine's mount point.

##### Example:

```hbs
// in route "foo.bar", where "bar" is an engine
{{link-to "^"}}        => application
{{link-to "^.foo"}}    => foo
{{link-to "^.^.boo"}}  => ERROR (see below for handling)
```

#### To Siblings

Linking into a sibling context will simply mean linking into the ancestor and then into its descendants. Since this results in tight coupling between contexts, it is recommended to declare explicit build dependencies between the various contexts whenever a sibling link is introduced.

##### Example:

```hbs
// current route is "foo.bar", where "bar" is an engine with
// a sibling engine "boo"
{{link-to "^.boo"}} => foo.boo
```

### Inter-Context Link Errors

If you attempt to link into a non-existent route or context, behavior should be similar to current behavior for linking to non-existent routes (e.g., error on render) but it should also include a note about context.

When routing to a non-existent context, the error should be similar to:

```bash
ERROR: Attempting to link to parent context, but one could not be found.
```

When routing to a non-existent route that lives in a different context, the error should be similar to:

```bash
ERROR: Attempting to link to route 'foo', but no route with that name exists in the context.
```

Exact wording is up for bikeshedding.

## Implementation and Other Considerations

### Resolving `^`

Since `^` is defined as a relative path character, we will need to introduce a unified way of resolving it to an absolute route path for use by the various linking constructs in Ember.

We propose introducing a new private method, potentially named `_resolveRouteName`, in the router that can handle this path resolution. Introducing this method at the router level will allow us to leverage the same functionality from `{{link-to}}` as well as programmatic constructs (e.g., `transitionTo`).

Implementation of this function can largely be based on `owner` information, assuming the `owner` has some knowledge of it's parent context. When interpreting the route path syntax, it would require walking the `owner` tree a number of levels equal to the number of `^` present in the route path and using the mount point present at that owner.

### Asynchronous Considerations

One primary concern is being able to link to engines that might not have been loaded yet. This design, leveraging the existing `{{link-to}}` functionality, means that those async issues (such as handling query params, loading/transition states, and knowing available, unloaded routes) should mostly "just work".

Since knowing available routes and transition information is an issue of the router, those problems can be solved at that level without changing this API. Additionally, [default query params](https://github.com/emberjs/rfcs/pull/113) and [dynamic segments](https://github.com/emberjs/rfcs/pull/120) both have relevant RFCs for handling their usage before the entire engine has loaded.

If, however, there is some need for this API to change to account for async Engines, it will also mean that it needs to be solved for all usages of `{{link-to}}` and is not a specific concern of linking between contexts.

### Testing

Testing an Engine that expects to link into another context currently would require mocking out those additional contexts for your tests. Even with documentation this process is complicated and can result in a lot of overhead for a potentially simple test. However, given that this would require the development of an entirely separate API from the one being proposed here and there is a working solution in place now, it seems best to handle the design of that API in an independent RFC devoted to mocking dependencies for Engine tests.

There should be no issues with unit and integration tests as those are already setup to not resolve routes since there is no router present.

# How We Teach This

Implementing this new micro-syntax, would require an addition to the Ember guides to discuss its behavior. However, since it is specific to applications that use engines, it can be introduced with those relevant guides; in fact, the `^` syntax should only be supported from within an Engine as there is no use case for it within an Application.

This new syntax is not critical to new users, only those that desire to use engines, and since engines themselves are new, we don't anticipate any major hurdles with educating existing users (as they will also need to reference the guides).

# Drawbacks

## Dual-Tree Concept

Introducing the `^` to routes means that we now need to know information about the context tree (e.g., the tree formed by the Engines in an Application) and the route tree. This adds a bit of conceptual overhead and a small amount of runtime overhead as well, but we anticipate the context tree should be, in almost all cases, small in scope (2 or maybe 3 levels).

## Expanded Route Micro-Syntax

Expanding the micro-syntax used by routes as a DSL introduces one more thing for developers to learn, but it is comparatively less than learning an entirely new construct for linking. This is explored in more depth in the "Alternatives" section.

# Alternatives

Two other approaches were considered for linking between different contexts.

## Other Types of Relative Paths

Using another syntax for relative pathing that would be more familiar to users (as opposed to `^`) was definitely a consideration. In particular, we considered using `../` or filesystem style traversals, since this would require little introduction for developers. This had two major issues though:

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

- ???
