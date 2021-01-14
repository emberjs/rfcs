---
Stage: Accepted
Start Date: 2020-01-14
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR:
---

# Deprecate jQuery Integration Optional Feature

## Summary

Deprecate the `jquery-integration` optional feature flag.

## Motivation

jQuery integration has been a part of Ember since the beginning. When Ember was
first released, a variety of browser features had not been standardized yet, and
jQuery was a necessary tool for creating code that worked in all major browsers.
This is no longer the case, and Ember is no longer internally reliant on jQuery.

That said, jQuery is still a useful library with many plugins, and that many
developers use. Adding jQuery to an Ember application is still completely
possible, and **not deprecated by this or any other RFC**.

The `jquery-integration` optional feature is not about removing jQuery from your
Ember application. It is instead about removing the _integration_ that Ember has
with jQuery, which is left over from the days when jQuery was used by Ember for
many different basic tasks. Specifically, disabling the feature removes the
following APIs:

- `Ember.$`, which is an alias for jQuery
- `this.$`, which is an alias for a jQuery selector that is scoped to `this`.
  This is available in Classic components and tests.

Deprecating the `jquery-integration` optional feature means that users will have
to transition away from using these specific APIs, but _not_ from using jQuery
in general. This will help us to clarify that Ember is not dependent on jQuery,
and allow us to simplify Ember's internals.

## Transition Path

This deprecation will be a 2 phase deprecation:

1. Deprecate the following optional feature flag settings:
  - `jquery-integration: true`
2. Deprecate specifying the optional feature at all

### Phase 1

The first phase will require that all users have toggled the optional feature to
`false`. This phase can be implemented immediately, and will position us to be
able to enter phase 2 in the next major version after implementation (currently
v4).

Since the not specifying the optional feature will cause it to be set to the
`true` by default in Ember v3, users will have to specify the optional feature
explicitly.

### Phase 2

Once a major version has been released after phase 1 has been implemented, all
users will be using the same setting for this optional feature, explicitly. In
the first release of Ember v4, we will be able to safely change the default
value of the optional feature to `false`. Users will be able to drop the
explicit setting at this point, and the blueprint will be updated to drop it as
well.

At this point, the default setting will be `false`, and explicitly setting it to
`false` will be allowed. Explicitly setting it to `true` will trigger an
assertion. This means that users can only possibly have one behavior specified,
and we can deprecate specifying the optional feature at all.

## How We Teach This

We should update the Optional Features guide to discuss in more detail how users
can migrate away from jQuery. In general, most usages should be fairly
straightforward, and the current guides do not cover it in great detail. They
also do not mention that `@ember/jquery` can be used without the feature flag.

### Updated Optional Feature Guide

jQuery is commonly used for event handling and many popular libraries for
charting and UI components. Ember was originally built using jQuery, but has
since refactored and is no longer dependent on it. jQuery can still be used
as in independent library alongside Ember, however.

There are a few APIs that exist in Ember still that directly expose jQuery,
Disabling this optional feature disables those APIs, but does not remove jQuery
itself. jQuery is provided by the `@ember/jquery` addon, independent of the
integration APIs.

#### Migrating Away from and disabling Ember jQuery integration APIs

To disable this feature, first migrate away from the jQuery integration APIs.
Below is a list of the jQuery specific APIs in Ember, and how to migrate away
from them.

- `this.$()` in Classic Ember components. This creates a jQuery selector that
  targets the component's element. You can migrate away by using `this.element`
  instead, which is the actual DOM element for the component. If you want to
  continue using jQuery via `@ember/jquery`, you can do so with `this.element`:

  ```js
  import $ from 'jquery';
  import Component from '@ember/copmonent';

  export default class MyComponent extends Component {
    didInsertElement() {
      let el = $(this.element);

      // ...
    }
  }
  ```

- Event handlers on Classic components, such as `click()` and `mouseEnter()`.
  These APIs still work without the integrations enabled, but they no longer
  receive a jQuery event, they receive a native Event instead. You can convert
  to using native events incrementally by using the [`ember-jquery-legacy`](https://github.com/emberjs/ember-jquery-legacy) addon, which provides a function that converts a jQuery
  event into a native event safely. Once all of your event handlers have been
  converted, you can disable jQuery integration.

- `Ember.$()`, which is an alias for the global jQuery. This can either be
  replaced with alternative, non-jquery based APIs, or by installing
  `@ember/jquery` and importing it directly.

- Global acceptance test helpers like `find()` or `click()`. These can be
  replaced with the `@ember/test-helpers`, which is the default for test helpers
  now.

- `this.$()` in component tests. This can be replaced with corresponding test
  helpers from `@ember/test-helpers`.

Note that if you disable these APIs, then all addons you use must also work
without them, as they will not be available at all.

Next, follow the instructions above to install `@ember/optional-features`, and
run the following command to change `@ember/optional-features`:

```sh
ember feature:disable jquery-integration
```

#### Including jQuery without integration APIs

If you would like to include jQuery without the Ember integration APIs, you can
install `@ember/jquery`:

```sh
ember install @ember/jquery
```

This will allow you to import jQuery from `jquery`:

```js
import $ from 'jquery';
```

#### Including jQuery with integration APIs

To include jQuery in your Ember app and enable the jQuery integration APIs such
as `this.$()`, follow the instructions above to install `@ember/optional-features`.
Next, enable the feature:

```sh
ember feature:enable jquery-integration
```

Then, install the `@ember/jquery` addon:

```sh
ember install @ember/jquery
```

Now, almost anywhere in your app, you can use the various jQuery integrations.

#### Removing jQuery completely

If you are working on an application that already has jQuery installed, and
would like to remove it, follow these steps.

First, refactor your own code to not depend on jQuery. See the section above on
how to do this. Keep in mind that you will have to remove jQuery usage entirely,
you cannot use solutions that replace the integration API with jQuery
independently such as `$(this.element)`.

Next, follow the instructions above to install `@ember/optional-features`, and
run the following command to change `@ember/optional-features`:

```sh
ember feature:disable jquery-integration
```

Then, remove `@ember/jquery` from your package.json.

This will remove jQuery from your vendor.js bundle and disable any use of jQuery
in Ember itself. Now your app will be about 30KB lighter!

### Deprecation Guide

Setting the `jquery-integration` optional feature to `true` has been
deprecated. You must set this feature to `false`, disabling jQuery integration.
This only disables **integration** with Ember, jQuery can still be included and
used as an independent library via the [`@ember/jquery` addon](https://github.com/emberjs/ember-jquery).

For more details on this optional feature, including the changes in
behavior disabling it causes and how you can disable it, see the
[optional features section](https://guides.emberjs.com/release/configuring-ember/optional-features/#toc_removing-jquery)
of the Ember guides.

## Drawbacks

- Could cause churn in some existing applications

## Alternatives

- We could continue supporting these jQuery integrations forever.

