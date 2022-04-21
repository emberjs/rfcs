---
Stage: Accepted
Start Date: 2022-04-21
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: 
---

<!--- 
Directions for above: 

Stage: Leave as is
Start Date: Fill in with today's date, YYYY-MM-DD
Release Date: Leave as is
Release Versions: Leave as is
Relevant Team(s): Fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies
RFC PR: Fill this in with the URL for the Proposal RFC PR
-->

# Public `RouterService` import

## Summary

This RFC proposes adding a public import for [`RouterService`](https://api.emberjs.com/ember/release/classes/RouterService)
making it extensible by addons, removing the external link assertion from `<LinkTo />` and rebuilding the `<LinkTo />` component
to rely entirely on the public routing service instead of the internal `-router`.

## Motivation

Today, users of [`ember-engines`](https://github.com/ember-engines/ember-engines) are required to utilize separate
custom components, helpers and services for linking. This leaves the experience feeling sub-par, in addition to introducing
extra maintenance friction attempting to keep these primitives aligned with those in `ember-source`.

By providing public a import for `RouterService`, `ember-engines` would be able to provide its own router service
implementation extending from the base implementation. Combined with minor changes to the engines-specfici behavior of
`link-to`,  this change would allow us to fill the existing DX gap when it comes to navigating between an Engine
and a host App *and* allow engines to eliminate use of the legacy components required to support 4.x versions of Ember.

These proposed changes would further allow `ember-engines` to deprecate the `<LinkToExternal />`
component provided by `ember-engines` package. As a result, there will be less
engine specific concepts and code would be more portable between regular applications and engines.

Once this RFC gets implemented, following will be possible in user defined Ember Engines:

* links to external routes in templates within the engine rendered via `<LinkTo />` component:

  ```hbs
  <LinkTo @route="external-route">Go Somewhere</LinkTo>
  ```

* using `RouterService` methods for redirects to the external routes:

  ```javascript
  # addon/components/my-component.js
  export default class extends Component {
    @service router;
    @action clickLink () {
      this.router.transitionTo('external-route');
    }
  }
  ```

**Bonus: improved DX for addons**

This RFC seeks to provide a solution that also solves another DX issue faced by engines developers.

Authors of large scale applications often create an addon(s) to contain reusable components that can
be rendered either in the context of the host App or Engine. In such scenarios an extra guard needs
to be introduced as the `<LinkToExternal />` component is available only in the context of an Engine
and does not exist in the context of regular applications, so it is common to see patterns such as the following:

```hbs
{{#if @isEngine}}
  <LinkToExternal @route="contact">Contact Us</LinkToExternal>
{{else}}
  <LinkTo @route="contact">Contact Us</LinkTo>
{{/if}}
```

Once this RFC gets implemented, above snippet could be replaced with

```hbs
<LinkTo @route="contact">Contact Us</LinkTo>
```

## Detailed design

### `ember-source` package

#### `<LinkTo />` component

The use of `-routing` service in `<LinkTo />` component would be replaced with the `router` service.
Currently, those properties being pulled from `-routing` service:

* `currentState`
* `currentRouteName`

Currently, those methods of `-routing` service used:

* `generateURL`
* `transitionTo`
* `isActiveForRoute`

Additionally any engines specific code or assertions for url and path generation would be removed, delegating
 these responsibilities instead to the router service. For instance, `<LinkTo />` should not be aware of `mountPoint`.

#### `RouterService`

Provide public import via `@ember/routing/router-service`.

#### Router Helpers

The changes outlined here improve the DX for engines for the not-yet implemented url helpers specified by [RFC#391](https://emberjs.github.io/rfcs/0391-router-helpers.html). Since those helpers will also utilize the routing service,
they would be compatible with both internal and external engines paths immediately.

### `ember-engines` package

While not the scope of this RFC to detail the exact specifics, once these changes are implemented, `ember-engines`
will provide a `router` service that extends from `@ember/routing/router-service`.

```javascript
# ember-engines/addon/services/router.js
import RouterService from '@ember/routing/router-service';

export default class extends RouterService {
  // ...
}
```

The extended `RouterService` would override or extend any necessary public methods and properties to provide a great DX
for engines developers.

## Prior Discussions

- [emberjs/rfcs#806](https://github.com/emberjs/rfcs/pull/806)
- [ember-engines/ember-engines#587](https://github.com/ember-engines/ember-engines/issues/587)
- [ember-engines/ember-engines#779](https://github.com/ember-engines/ember-engines/issues/779)

## How we teach this

Ember Engines documentation provided at [ember-engines.com](https://ember-engines.com)
should be updated with `<LinkTo />` replacing all the references of `<LinkToExternal />` component.

Transition path needs to be published on [ember-engines.com](https://ember-engines.com) containing following steps:

1. Replace use of `<LinkToExternal @route="external-route" />` component
   with `<LinkTo @route="external-route" />` in every engine codebase.
2. Users using `ember-engines-router-service` would need to replace any of it's usage with
   the `RouterService` provided by `ember-engines`. This addon was created to solve part of
   this problem previously, and would likely be updated to become a true polyfill for the ember-engines
   routing service.

## Drawbacks

The removal of assertions within `<LinkTo />` specific to ensuring that external paths are properly mapped means that
it is now the responsibility of the routing service provided by ember-engines to enforce that encapsulation is not
broken unintentionally, since a user can now provide any fully qualified path to `<LinkTo />`.

A similar issue arises with the routing service's `transitionTo` method when invoked with a url. For instance:

```ts
router.transitionTo('/');
```

Some prior discussion on this point is [viewable here](https://github.com/ember-engines/ember-engines/issues/587#issuecomment-483531420).
In the case of this RFC, it would similarly become the responsibility of the engine's routing service to prefix such a url with the
engine's mount path as appropriate, or decide to error, to preserve encapsulation.

In practice we feel the flexibility and improved DX of this solution outweighs these drawbacks, and intend to
continue enforcing the same constraints via the engines provided service.

## Alternatives

* Utilize `@ember/legacy-built-in-components` to keep `<LinkToExternal />`
  a thin wrapper around legacy implementation of `<LinkTo />` component.

  As this may be considered temporary solution to solve Ember 4 compatibility for `ember-engines`,
  this is not viable long term solution as any changes made to `<LinkTo />` component in the Ember.js codebase
  will need to be backported to `@ember/legacy-built-in-components`.

* Re-implement `<LinkTo />` functionality in `ember-engine`.

  This is not viable long term solution as any changes made to `<LinkTo />` component in the Ember.js codebase
  would need to be backported to `ember-engines`.

* Deprecate and remove `externalRoutes` mapping.

  This would require providing *full route name* to `<LinkTo />` component *including* engine mount point
  within the engine code.
  This option is not viable as it breaks isolation principle of Ember Engines
  whereas every engine should be self-scoped micro-frontend.

* Introduce `{{path-for}}` Engine Router helpers.

  This option was discussed in [RFC #806 "Engine Router Helpers"](https://github.com/emberjs/rfcs/pull/806)
  which was considered as partial solution to teh problems discussed in this RFC.
  
* Make ember-source handle all of the engine specific logic for routing itself.

  To this point engines have only minimally been accounted for in the code of ember-source, this would digress from that.
  This integration could be done at a later time, but it seems unnecessary to currently hoist engines specific logic
  into applications that may not have any engines.

## Unresolved questions

None.
