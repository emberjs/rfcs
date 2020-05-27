- Start Date: 2020-04-23
- Relevant Team(s): Ember.js, Learning
- RFC PR: [#619](https://github.com/emberjs/rfcs/pull/619)
- Tracking: (leave this empty)

# Better environment handling

## Summary

This RFC proposes to add a `TESTING` export to `@glimmer/env`, to allow to more easily work with common environment questions.
Additionally, it proposes to add the `@glimmer/env` module to the [API docs](https://api.emberjs.com/), like `@glimmer/tracking`.

## Motivation

Currently, there is no _one_ way to identify if the application is running in test mode.

Some reasons why you would want to check for that in your code include:

- Disabling polling code in tests
- Preventing test-breaking code like window.reload()
- Preventing log output in tests

There are three main ways to solve this, as of now:

1. `Ember.testing`

This is not really recommended anymore. There is no module export for this (you need to use the `ember` import for it).

It cannot be used in module scope at all (see [no-ember-testing-in-module-scope eslint-plugin-ember rule](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-ember-testing-in-module-scope.md)).

2. `import config from 'my-app/config/environment` && `config.environment === 'test'`

This only works inside of an app. Addons cannot use this approach, as they don't know the application name.

3. `getOwner(this).resolveRegistration('config:environment').environment === 'test'`

This requires access to the container, and is also a rather complicated syntax for such a thing.
It might not be immediately clear what exactly is happening here,
and the `resolveRegistration()` syntax is probably not commonly used by many developers.

This RFC wants to solve this by providing a single, unified & community-approved solution to this problem.

It also wants to provide an easy solution to test production builds of an application.

## Detailed design

```js
import { TESTING } from "@glimmer/env";
```

This is a constant (does not change / live update without a refresh).

It is true in the following cases:

- `ember test`
- `ember test --environment=production`
- `ember serve` & `http://localhost:4200/tests`

It is false in all other cases, like these:

- `ember serve` & any other path
- `ember build --environment=production`

In addition, the `@glimmer/env` module should be added to the [Ember API docs](https://api.emberjs.com/), like `@glimmer/tracking`.

Finally, we should deprecate these older functions/modules:

- `Ember.testing`
- `runInDebug` from `@ember/debug`

We will deprecate them immediately, as the implementation of TESTING will work retroactively - anyone that can use ember-cli-babel@7 would be able to use it, regardless of ember-source version.

## How we teach this

In addition to adding this to the Ember API docs, a sub page should be added to the Ember.js Guides. This could either live under "Application concerns" or under "Configuration > Configuring your app".

## Drawbacks

It slightly extends API surface.

## Alternatives

We could also do nothing and propose users to continue using `Ember.testing` or a home-grown solution instead. However, I think that these problems are common enough to warrant a nice, built-in solution for them.

We can also encourage users to use e.g. `getOwner(this).resolveRegistration('config:environment').environment === 'test'`, however this will not work in plain classes without Owner access.

We can also encourage users to use `import config from 'my-app/config/environment'`, however that does not work in addons.

## Unresolved questions

-
