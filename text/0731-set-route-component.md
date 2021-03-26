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

## Detailed design

This RFC proposes adding the `setRouteComponent` API, importable from
`@ember/routing/route`.

```js
declare function setRouteComponent(component: Component, route: Route): void;
```

Users can associate any component definition with a route via
`setRouteComponent`. When a component has been associated with a route this way,
the route will render this component at its root rather than rendering the route
template defined in `app/templates/routes`. The component will be passed the
`@model` and `@controller` arguments, which correspond to the model returned
from the route's `model` hook and the instance of the route's controller.

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

## How we teach this

For the time being, this API will not be the recommended way of writing Ember
apps. As such, it will not be included in the guides.

### API Docs

`setRouteTemplate` can be used to associate a component with a route. When
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

## Drawbacks

The primary drawback is that this new API increases incoherence in the short
term, since there will now be multiple ways to define templates for routes.

## Alternatives

- We could introduce an API for setting the template of a route directly,
  `setRouteTemplate`. This would still require users to use controllers as the
  backing context for the template, and the API becomes a bit awkward at that
  point - should you associate the template with the route, or the controller?
