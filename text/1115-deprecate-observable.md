---
stage: accepted
start-date: 2025-06-16T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
  - data
  - learning
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1115
project-link:
---

# Deprecate Observers

## Summary

Deprecate the `Observable` Mixin, `observer` helper, and public observer
functionality, including:

- The `Observable` mixin (`@ember/object/observable`) and the methods it adds
  to `EmberObject` and its descendants:
  - `get`, `set`, `getProperties`, `setProperties`
  - `notifyPropertyChange`
  - `addObserver`, `removeObserver`
  - `incrementProperty`, `decrementProperty`, `toggleProperty`
  - `cacheFor`
- The `observer` helper from `@ember/object`
- The standalone `addObserver` and `removeObserver` functions from
  `@ember/object/observers`

The standalone `get`, `set`, `getProperties`, `setProperties`, and
`notifyPropertyChange` functions from `@ember/object` are not deprecated by
this RFC.

## Motivation

Using observers has been soft-deprecated ever since the introduction of
tracking. The guides do not teach them, and eslint-plugin-ember's
[`no-observers`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-observers.md)
rule has recommended against them for years. Removing support for them will
significantly clean up the code base. (Note that for us to entirely remove
observer support internally, we will likely need to deprecate other
functionality in addition to what is included here.)

Additionally, Ember has also not recommended the use of Mixins for a while.
Since `Observable` uses the Mixin architecture, it will also need to be
deprecated for us to fully remove Mixins.

## Transition Path

A full deprecation guide is PR'd here: https://github.com/ember-learn/deprecation-app/pull/1407

The deprecations will target `until: '8.0.0'`.

### Observable Mixin

Some of the methods provided by `Observable` are available as standalone methods,
for these the transition path is straight-forward:

* `get` -> Import from `@ember/object`
* `getProperties` -> Import from `@ember/object`
* `set` -> Import from `@ember/object`
* `setProperties` -> Import from `@ember/object`
* `notifyPropertyChange` -> Import from `@ember/object`

The standalone functions are only needed when interoperating with legacy
computed properties and Ember's proxies. Rather than adopting them long-term,
most code should refactor `@computed` properties to native getters over
`@tracked` state, at which point plain property access and assignment work.

For sugar helpers they can easily be replaced with a bit more verbosity:

* `incrementProperty`
    ```js
    this.count += 1; // Tracking
    set(this, 'count', get(this, 'count') + 1); // Legacy 
    ```
* `decrementProperty`
    ```js
    this.count -= 1; // Tracking
    set(this, 'count', get(this, 'count') - 1); // Legacy 
    ```
* `toggleProperty`
    ```js
    this.flag = !this.flag; // Tracking
    set(this, 'flag', !get(this, 'flag')); // Legacy
    ```

For others, there will be no replacement and code that depends on them
will have to be refactored:

* `addObserver` (see below)
* `removeObserver` (see below)
* `cacheFor` — for memoization, use native getters with
  [`@cached`](https://api.emberjs.com/ember/release/functions/@glimmer%2Ftracking/cached)
  from `@glimmer/tracking`. There is no replacement for peeking at a cached
  value without computing it.

### Observers

The `observer` helper does not work with native class syntax and as such,
shouldn't be relied upon. It will not be replaced, and neither will the
standalone `addObserver` and `removeObserver` functions. The transition path
depends on what the observer was doing.

Observers that derive state from other state should become native getters over
`@tracked` properties. The derivation then happens lazily when the value is
read instead of eagerly when a dependency changes.

Observers that perform side effects should be refactored so the side effect
happens explicitly at the site of the mutation, for example in the action that
performs the change.

Observers used in classic components to react to argument changes, often to
drive a third-party library, should become
[modifiers](https://guides.emberjs.com/release/components/template-lifecycle-dom-and-modifiers/),
which re-run when their arguments change and are tied to an element's
lifecycle.

Some libraries use observers to find out when tracked state has changed
outside of rendering, for example ember-concurrency's `waitFor`, and patterns
in ember-animated, ember-data, and ember-power-select. Today the only
workarounds are polling via `requestAnimationFrame` or timers. Ember should
expose a deliberately low-level API for consuming tracked state outside the
renderer, likely building on
`createCache` / `getValue` with a way to be notified of invalidation.
Designing that API is left to a follow-up RFC, but the observer APIs
deprecated here should not be removed until it exists.

### Ecosystem impact

- eslint-plugin-ember already ships `no-observers`, so no new lint rules are
  needed. Its docs can link to the deprecation guide once it ships.
- Addons that use observers (ember-concurrency, ember-animated,
  ember-power-select, parts of ember-data) will need to migrate. The
  `until: '8.0.0'` window is intended to give them time.
- The published types for `@ember/object/observable`,
  `@ember/object/observers`, and `observer` will be marked `@deprecated` and
  removed along with the APIs.

## Exploration

To validate this deprecation, I've tried removing this functionality in the following PRs:
* Inline Observable Mixin: https://github.com/emberjs/ember.js/pull/20928
* Remove `observer` helper and inlined Observable methods: https://github.com/emberjs/ember.js/pull/20937

## How We Teach This

We should remove all references to this functionality from the guides and mark
the relevant API docs as deprecated. Observers are not part of the modern
learning path, so no re-organization is needed.

The deprecation guide (PR'd above) covers each use case, showing both the
incremental migration for legacy `EmberObject` / `@computed` code and the
final native-class form. The deprecation should also be mentioned in the
release blog post.

## Drawbacks

Many users probably rely on this functionality. However, it's almost certainly
something that we don't want to support long-term.

Non-rendering integration with the reactivity system has no efficient
replacement until the low-level API described above exists. The deprecation
can ship first, but removal must wait for that API.

## Alternatives

* Just inline this functionality so that we can at least get rid of Mixins.
* Keep the standalone `addObserver`/`removeObserver` functions as the
  low-level integration API. Rejected because they expose the legacy eager
  notification model rather than an autotracking primitive.
* Do nothing, which blocks removing Mixins and means maintaining two
  reactivity systems indefinitely.

## Unresolved questions

The design of the low-level API for consuming tracked state outside the
renderer is left to a follow-up RFC. This RFC takes the position that the
deprecated APIs are not removed until that API ships.
