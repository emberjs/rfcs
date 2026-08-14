---
stage: accepted
start-date: 2026-08-14T00:00:00.000Z
release-date:
release-versions:
teams:
  - cli
  - framework
  - learning
  - steering
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
---

# Deprecation Staging

## Summary

Establish a two-stage system for deprecations in `ember-source` that allows
them to be rolled out incrementally. The two stages are:

1. **Available** — the deprecation exists but is only active for apps that
   explicitly opt in.
2. **Enabled** — the deprecation is active for everyone and cannot be
   disabled.

This replaces the design proposed in [RFC #0649](./0649-deprecation-staging.md)
with a significantly simpler model that discards the complex addon-compliance
machinery in favour of a single opt-in config that lives in the standard build
entry-point for both Vite-based and classic ember-cli projects.

## Motivation

RFC 0649 identified real problems with the current deprecation system:

* **Ecosystem absorption** — addons may still use a deprecated API, so an app
  that has already removed the usage will keep seeing the warning. There is
  currently no way to introduce a deprecation for early adopters while the
  addon ecosystem catches up.

* **Pacing** — once a deprecation RFC is merged there are no checkpoints to
  slow or speed up the rollout based on real-world feedback.

* **Early adoption** — because deprecations are conservative and all-or-nothing
  today, early adopters have no strong signal that an API is on the way out
  before they are forced to act.

RFC 0649 addressed these problems but the proposed solution — a multi-package
compliance graph with `stage`, `version`, and `"auto"` declarations threaded
through every addon's `package.json` — proved too complex to implement fully
and too hard to explain to the community. The subset of that RFC that _was_
shipped (`for` and `since` on the `deprecate()` call) is the right foundation;
this RFC builds on it.

The key insight that simplifies the design is:

> *Available* deprecations are an opt-in early-warning signal, not a compliance
> contract. *Enabled* deprecations are universal and need no configuration at
> all.

With this framing there is no need for cross-package compliance propagation,
build-time errors about mismatched addon versions, or any notion of
"declaring compliance". Apps that want early warnings opt in; everyone else
sees nothing until the deprecation is enabled.

## Detailed design

### Deprecation stages

#### Available

A deprecation enters the **available** stage when:

1. The replacement API is stable and the deprecated API is fully covered.
2. The Ember team wants early adopters and addon authors to begin migrating
   before the deprecation is visible to all users.

Available deprecations are **silent by default**. They are only logged (or
thrown, if `RAISE_ON_DEPRECATION` is set) for apps that explicitly opt in.

#### Enabled

A deprecation advances to the **enabled** stage when the team is confident
that the addon ecosystem has absorbed it and it is appropriate for all users
to act on it.

Enabled deprecations **cannot be disabled** and are always logged regardless
of any opt-in configuration.

### The `deprecate()` function

RFC 0649 already added `for` and `since` to `DeprecationOptions`. This RFC
keeps that shape unchanged:

```ts
interface DeprecationOptions {
  id: string;
  until: string;
  url?: string;
  for: string;       // package name, e.g. 'ember-source'
  since: {
    available: string;  // SemVer string, required
    enabled?: string;   // SemVer string, set when promoted to enabled
  };
}
```

A deprecation that only has `since.available` is in the **available** stage.
Once `since.enabled` is also populated the deprecation is in the **enabled**
stage.

If the `since` key is absent (legacy deprecations added before this RFC) the
deprecation is treated as if it were in the **enabled** stage.

Example of an available deprecation:

```js
deprecate('`isVisible` is deprecated. Use CSS directly instead.', false, {
  id: 'ember-source.component.is-visible',
  for: 'ember-source',
  since: { available: '6.5.0' },
  until: '7.0.0',
  url: 'https://deprecations.emberjs.com/id/ember-source.component.is-visible',
});
```

Once it advances to enabled:

```js
deprecate('`isVisible` is deprecated. Use CSS directly instead.', false, {
  id: 'ember-source.component.is-visible',
  for: 'ember-source',
  since: { available: '6.5.0', enabled: '6.8.0' },
  until: '7.0.0',
  url: 'https://deprecations.emberjs.com/id/ember-source.component.is-visible',
});
```

### Opting into available deprecations

Available deprecations from `ember-source` are opt-in only. An application
opts in by listing the package (or packages) it wants to receive available
deprecations from in its build configuration.

This RFC scopes the opt-in mechanism to `ember-source`. Third-party packages
may adopt the same `for`/`since` convention on their `deprecate()` calls, but
the tooling for opting into those packages is left to a future RFC.

#### Vite / Embroider 4 projects

In a project using `@embroider/vite` (the default for new projects), the
opt-in lives in `vite.config.ts` (or `vite.config.js`):

```ts
import { defineConfig } from 'vite';
import { ember } from '@embroider/vite';

export default defineConfig({
  plugins: [
    ember({
      availableDeprecations: ['ember-source'],
    }),
  ],
});
```

#### Classic ember-cli projects

In a project using `ember-cli-build.js`:

```js
'use strict';
const EmberApp = require('ember-cli/lib/broccoli/ember-app');

module.exports = function (defaults) {
  const app = new EmberApp(defaults, {
    availableDeprecations: ['ember-source'],
  });
  return app.toTree();
};
```

### Runtime behaviour

At build time, the opt-in list is compiled into a module that the deprecation
runtime can import. At runtime, when `deprecate()` is invoked:

1. If `since.enabled` is present → always log/assert regardless of config
   (enabled stage).
2. If only `since.available` is present → log/assert only if
   `deprecation.for` appears in the compiled opt-in list (available stage,
   opted in).
3. If `since` is absent → treat as enabled (legacy compatibility).

The runtime check is a simple set-membership test and has negligible
performance impact.

### New app blueprints

Newly generated apps will **not** include `availableDeprecations` by default.
An empty opt-in list is the correct default: new projects should not receive
noise from deprecations that have not yet been enabled for everyone.

Addon blueprints are unchanged. Addon authors are encouraged to opt their
test apps into `availableDeprecations: ['ember-source']` so they get early
signals, but this is not enforced by the tooling.

### Relationship to existing deprecation tooling

* **`registerDeprecationHandler()`**: unchanged. Available deprecations that
  are opted in will flow through the handler in exactly the same way as
  enabled ones.

* **`Ember.ENV.RAISE_ON_DEPRECATION`**: unchanged. When set, opted-in
  available deprecations (and all enabled ones) will throw instead of warn.

* **`ember-cli-deprecation-workflow`**: this is a separate, optional addon
  for managing the workflow of resolving deprecations one at a time. Nothing
  in this RFC conflicts with it or changes how it works. Developers may
  continue to use it alongside the opt-in mechanism described here.

## How we teach this

### Mental model

The core concept is straightforward:

> Deprecations are either *available* (opt-in) or *enabled* (everyone sees
> them). You will never be surprised by a new wall of deprecation warnings
> after an upgrade unless you have explicitly asked for them.

### Messaging in deprecation warnings

Available deprecations that are shown (because the app opted in) should
include a note in the warning message, for example:

> This deprecation is in the *available* stage. You are seeing it because your
> project has opted in. It will become *enabled* (visible to all) in a future
> release.

Enabled deprecations need no extra annotation because they behave exactly as
deprecations do today.

### Documentation updates

The following pages will need to be updated:

- https://cli.emberjs.com/release/writing-addons/deprecations/
- https://guides.emberjs.com/release/configuring-ember/handling-deprecations/
- https://deprecations.emberjs.com — each deprecation entry should clearly
  show whether it is currently *available* or *enabled*, along with the version
  in which it reached each stage.

### Blueprint changes

The generated `vite.config.ts` and `ember-cli-build.js` should include a
comment pointing to the documentation for `availableDeprecations`, even when
the list is empty, so developers know the feature exists:

```ts
ember({
  // To receive early deprecation warnings before they are enabled for everyone,
  // add 'ember-source' to this list.
  // See: https://guides.emberjs.com/release/configuring-ember/handling-deprecations/
  // availableDeprecations: ['ember-source'],
})
```

## Drawbacks

* **No enforcement / compliance contract** — unlike RFC 0649, there is no way
  for an app to declare that it is *compliant* with a set of deprecations and
  have the tooling enforce that contract across its dependency tree. Apps that
  opt in will see warnings; they cannot yet turn those warnings into hard
  errors for specific packages only (beyond `RAISE_ON_DEPRECATION`, which is
  global).

* **Addon authors must opt in themselves** — addon test suites will not
  receive available deprecation warnings by default. Addon authors must
  remember to set `availableDeprecations: ['ember-source']` in their test
  app's build config.

## Alternatives

### RFC 0649 (full compliance system)

The original RFC proposed a comprehensive compliance-declaration system where
every addon and app could declare which version of which package it was
compliant with, and Ember would propagate these declarations and enforce them
at build time. This approach was not fully implemented because the complexity
was hard to justify for the benefit it provided. This RFC preserves only the
parts that shipped (`for` and `since` on `deprecate()`) and replaces the
unimplemented parts with a simpler mechanism.

### Keep deprecations in `package.json`

RFC 0649 stored configuration in the `ember` key of `package.json`. This RFC
moves it to the build entry-point (`vite.config.ts` or `ember-cli-build.js`)
for two reasons:

1. Build configuration naturally belongs next to other build configuration.
   The `ember` key in `package.json` is not a well-known convention and
   requires special tooling to process.
2. Vite-based projects have no `ember-cli-build.js`, so a `package.json` key
   would be the only shared location — but `package.json` is a poor choice
   for configuration that only affects development builds.

### A dedicated `config/ember-deprecations.js` file

A standalone file was considered but rejected to avoid confusion with
`ember-cli-deprecation-workflow`, which already has its own configuration
convention. Inline build config is discoverable and co-located with other
build options.

## Unresolved questions

* Should `availableDeprecations` support a minimum-version qualifier, e.g.
  `{ package: 'ember-source', since: '6.5.0' }`, so that apps can opt into
  only the deprecations that were available as of a particular release? This
  would let an app say "show me everything available in 6.5 and later" rather
  than all available deprecations from the package.

* Should there be a way to opt into available deprecations from packages other
  than `ember-source` in this RFC, or should that be deferred?
