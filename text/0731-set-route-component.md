---
Stage: Accepted
Start Date: 2021-03-25
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Framework
RFC PR: https://github.com/emberjs/rfcs/pull/731
---

# Add `setRouteComponent`

## Summary

Adds an API for associating a component with a route. When used, the associated
component will be rendered instead of the route's template, allowing Ember
apps to use new features such as template imports and strict mode in whole
applications.

```js
import Route, { setRouteComponent } from '@ember/routing/route';
import MyRouteComponent from './component';

export default class MyRoute extends Route {}

setRouteComponent(MyRouteComponent, MyRoute);
```

## Motivation

Recently, Ember added support for strict mode, which enables the usage of
template imports:

```js
import Component from '@glimmer/component';
import { setComponentTemplate } from '@ember/component';
import { precompileTemplate } from '@ember/template-compilation';
import MyButton from './my-button';

export default class Example extends Component {}

setComponentTemplate(
  precompileTemplate(
    `<MyButton/>`,
    {
      strictMode: true,
      scope() {
        return { MyButton };
      }
    }
  ),
  Example
);
```

This example demonstrates how to use the new low level APIs to do this. Clearly,
this isn't the final API that Ember users should write in their apps, as it's
very verbose overall, but it's the first step toward defining a new
programming model which will come together in a future edition. We have,
essentially, entered the pit of incoherence, which means that some of these
features will not be fully complete or ready for mainstream use. Template
imports are such a feature, partially because we don't have a final authoring
format, but also partially because you cannot use them currently with a very
important part of Ember: Routes.

There currently is not a way to directly set the template of a route the same
way that you can set the template of a component with `setComponentTemplate`.
This means you cannot associate a template defined with `precompileTemplate`
with a route, and so there is no way to use imported values with them. Instead,
components must still be defined in the `app/components` directory, and then
used via resolution in a route's template defined in `app/templates/routes`.

This RFC seeks to add a method for associating a _component_ with a route. This
will allow users to associate strict mode templates with a route via a
component - either a template only component or one with a backing class. This
will allow the community to continue experimenting with different template
import formats more thoroughly, and will also allow users to experiment with
building Ember apps without controllers by effectively bypassing them.
Controllers will not be fully removed, as they still have some functionality
which has not been replaced, but this will help to see where those gaps are and
figure out what is needed to fill them.

This takes us one step further into the pit of incoherence, with the goal of
better defining how to move forward in two major areas that we are aiming for in
the next peak of coherence.

### Exploration directions unlocked by this API

There are a number of exploration directions that are unlocked by this API,
including:

- **Alternatives to controllers**: As mentioned above, this API will allow users
  to explore building Ember apps without using controllers. Removing controllers
  has been an agenda item on the roadmap for some time, they can be confusing to
  the modern Octane mental model and are a stumbling block for new users.
- **Alternative directory structures**: Allowing routes to render components
  will also unlock exploring colocating templates with routes in various ways.
  Components could be defined in the `app/routes` directory alongside routes,
  or even potentially in the same file, using the strict mode primitives.
- **More flexible testing setups**: Allowing route to render a component may
  also have the advantage of allowing that component integration-tested without
  the need to boot an entire Ember app or require the setup of a router. This
  could lead to better testing patterns in the future.
- **Alternative learning paths and structures**: In other frontend frameworks
  such as React, components are used in conjunction with a router to render
  pages – supporting similar functionality would allow a workflow familiar
  to developers coming from other disciplines, and may make it easier for us
  to teach Ember to a wider audience.

### History with Routable Components

This RFC mirrors a past RFC to make [components routable](https://github.com/ef4/rfcs/blob/routeable-components/active/0000-routeable-components.md).
Routeable components was an RFC that was first opened during the Ember v1
era, and it's scope was very large. At the time, Ember lacked a lot of the
foundational primitives that have since been added to the framework, and each
potential change in the router API required many other changes in other parts
of the framework overall. There were many other initiatives at the time
(angle bracket components, named arguments, named blocks), so it ultimately was
not finished. However, the idea was still considered a good idea, and the hope
was that we would return to it at some point in the future.

This RFC is the first step toward returning to the ideas behind "routeable
components". Unlike the original proposal, however, it is intentionally limited
in scope and focuses on adding a single piece of specific functionality. In
time, we can continue to add more functionality, and eventually make using
components with routes the default.


## Detailed design

This RFC proposes adding the `setRouteComponent` API, importable from
`@ember/routing/route`.

```js
declare function setRouteComponent(component: Component, route: Route): Route;
```

Users can associate any component definition with a route via
`setRouteComponent`. When a component has been associated with a route this way,
the route will render this component at its root rather than rendering the route
template defined in `app/templates/routes`. The route's `render` and
`renderTemplate` hooks will also be ignored, and will not be run as they would
have previously. For convenience, `setRouteComponent` will return the route
class (the second argument).

The component will be passed the `@model` and `@controller` arguments, which
correspond to the model returned from the route's `model` hook and the instance
of the route's controller.

This will effectively be syntactic sugar for rendering a top-level component
within a route template today:

```js
import Route, { setRouteComponent } from '@ember/routing/route';
import MyRouteComponent from './component';

export default class MyRoute extends Route {}

setRouteComponent(MyRouteComponent, MyRoute);
```

Is equivalent to having the following in the `my-route.hbs` file:

```hbs
<MyRouteComponent @model={{@model}} @controller={{this}} />
```

Assuming that `MyRouteComponent` is resolvable via that name.

No changes will be made to the route or controller lifecycle, hooks, or
behaviors in general, since this is not related to the goal of unlocking
experimentation with strict mode and route templates, or the goal of
experimenting with controller-less applications. Changes in these areas will be
done in future RFCs instead.

### Component lifecycle

The lifecycle of the component will be the same as if it were a top-level
component rendered in a route template today, as in the example above. Timing
semantics surrounding rerender, and switching to different models and routes,
will be the same, as will the behavior of the `{{@model}}` value.

### `@controller` argument details

The `@controller` argument will _lazily_ access the route's controller. In the
future, this controller _may_ be instantiated lazily as well, and only created
upon first access.

In addition, for cross-compatibility and to ease transition in the future,
`@controller` will be made available in standard route templates. It will be
an alias for `{{this}}`, effectively.

### Routing substates

Routes currently can defer to the `loading` and `error` substates based on the
current state of the `model()` hook. These substates are _themselves_
implemented as routes - you can define a `routes/loading.js` and
`controller/loading.js` to go along with the `app/templates/routes/loading.js`
for a particular route. So, the `setRouteComponent` API will work in the same
way with these routes, and allow you to associate a component to render in
them.

## How we teach this

For the time being, this API will not be the recommended way of writing Ember
apps. As such, it will not be included in the guides.

### API Docs

`setRouteComponent` can be used to associate a component with a route. When
associated, this component will be rendered instead of the route's template. The
component receives the `@model` and `@controller` arguments,  which correspond
to the model returned from the route's `model` hook and the instance of the
route's controller.

```js
import Route, { setRouteComponent } from '@ember/routing/route';
import MyRouteComponent from './component';

export default class MyRoute extends Route {}

setRouteComponent(MyRouteComponent, MyRoute);
```

Is equivalent to having the following in the `my-route.hbs` file:

```hbs
<MyRouteComponent @model={{@model}} @controller={{this}} />
```

Assuming that `MyRouteComponent` is resolvable via that name.

`setRouteComponent` can be used with any route class, including route classes
defined for `loading` and `error` substates.

## Drawbacks

The primary drawback is that this new API increases incoherence in the short
term, since there will now be multiple ways to define templates for routes.

## Alternatives

- We could introduce an API for setting the template of a route directly,
  `setRouteTemplate`. This would still require users to use controllers as the
  backing context for the template, and the API becomes a bit awkward at that
  point - should you associate the template with the route, or the controller?
