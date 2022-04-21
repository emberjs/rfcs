---
Stage: Accepted
Start Date: 
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

This RFC proposes adding public import for [`RouterService`](https://api.emberjs.com/ember/release/classes/RouterService)
making it extendable by addons.

## Motivation

By providing public import for `RouterService` and making it extendable,
[`ember-engines`](https://github.com/ember-engines/ember-engines) would provide extended version
of the `RouterService` to the user defined Ember Engines.
This change allows to fulfill the existing gap when it comes to navigating between Ember Engine and host app
*and* solve Ember v4 compatibility issue that exists in `ember-engines` today.

Proposed changes would allow to deprecate `<LinkToExternal />`
component provided by `ember-engines` package. As a result, there will be less
engine specific concepts and code would be portable between regular app and Ember Engines.

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

There is another problem this RFC seeks to solve.
Authors of large scale applications often create an addon(s) to contain reusable components
that can be rendered either in the context of the host app or Ember Engine.
In such scenario an extra guard needs to be introduced as `<LinkToExternal />` component
available only in the context of Ember Engine and does not exist in the context of regular application,
so it's common to see such pattern:

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

The use of `-routing` service in `<LinkTo />` component needs to be replaced with `router` service.
Currently, those properties being pulled from `-routing` service:

* `currentState`
* `currentRouteName`

Currently, those methods of `-routing` service used:

* `generateURL`
* `transitionTo`
* `isActiveForRoute`

#### `RouterService`

Provide public import via `@ember/routing/router-service`.

### `ember-engines` package

Provide `router` service that extends from `@ember/routing/router-service`:

```javascript
# ember-engines/addon/services/router.js
import RouterService from '@ember/routing/router-service';

export default class extends RouterService {
  // ...
}
```

The extended `RouterService` should override make all public methods and properties play nice with Ember Engines citizen.

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## How we teach this

Ember Engines documentation provided at [ember-engines.com](https://ember-engines.com)
should be updated with `<LinkTo />` replacing all the references of `<LinkToExternal />` component.

Transition path needs to be published on [ember-engines.com](https://ember-engines.com) containing following steps:

1. Replace use of `<LinkToExternal @route="external-route" />` component
   with `<LinkTo @route="external-route" />` in every engine codebase.
2. If `ember-engines-router-service` was installed and used, replace any of it' usage with
   `RouterService` provided by `ember-engines`. This should be done in the same PR
   updating `ember-engines` to the version that ships `RouterService`.

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

## Unresolved questions

None.
