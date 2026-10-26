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

Deprecate the observer APIs:
the `observer` helper, `addObserver` and `removeObserver`
(both the `@ember/object/observers` functions and the `EmberObject` methods),
and the `Observable` mixin.
The other methods that `Observable` adds to `EmberObject`, such as `get` and `set`,
are deprecated along with `EmberObject`
by [RFC 1234](https://rfcs.emberjs.com/id/1234-deprecate-ember-object).

## Motivation

Observers have not been recommended since Octane replaced them with autotracking,
and eslint-plugin-ember's
[`no-observers`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-observers.md)
rule has flagged them for years.
Removing them shrinks Ember's internals (see [Exploration](#exploration)).

`Observable` is a mixin, so
[RFC 1116](https://rfcs.emberjs.com/id/1116-deprecate-mixins)
cannot remove mixins until `Observable` is deprecated.

## Transition Path

Each deprecated API logs a deprecation when it is called,
or, for `Observable`, when the mixin is applied.
The deprecations target `until: '8.0.0'`.

No API directly replaces observers.
The fix depends on what the observer does,
and the [deprecation guide](#deprecation-guide) has an example of each:

| Observer used to… | Replace with |
| ----------------- | ------------ |
| derive state from other state | a native getter over `@tracked` properties |
| perform a side effect | an explicit call where the change happens |
| react to classic component arguments | a [modifier](https://guides.emberjs.com/release/components/template-lifecycle-dom-and-modifiers/) |
| watch state outside rendering | nothing yet (see below) |

### Integration Outside Rendering

Some libraries use observers to learn when state changes outside rendering.
ember-concurrency's [`waitForProperty`](https://github.com/machty/ember-concurrency/blob/0ffd122ed65f68970e0535ca2e54779c0140cac7/packages/ember-concurrency/src/-private/wait-for.js#L97-L119)
re-checks its predicate each time the observed property changes.
ember-animated uses observers in its task
[`.observes()`](https://github.com/ember-animation/ember-animated/blob/f0a53fb6c947f9018129dc1b97e21339397612ed/addon/src/-private/ember-scheduler.ts#L76-L85) modifier
and in [`<AnimatedEach>`](https://github.com/ember-animation/ember-animated/blob/f0a53fb6c947f9018129dc1b97e21339397612ed/addon/src/components/animated-each.gts#L222-L241),
and EmberData's [Inspector data adapter](https://github.com/emberjs/data/blob/a8ac10f9b2f8e2280b44fed40dedf5fb872cd5f4/packages/debug/src/data-adapter.ts#L739-L758)
observes records to push updates to the Ember Inspector.

Autotracking has no public way to do this.
`createCache` and `getValue` from `@glimmer/tracking/primitives/cache`
memoize a function over tracked state,
but nothing tells the caller when that result becomes stale;
only the renderer gets that signal.
A follow-up RFC should add a low-level API for that notification,
so that, for example, ember-concurrency can resume a waiting task
when the value changes.

### Ecosystem Impact

- eslint-plugin-ember already ships `no-observers`,
  so no new lint rules are needed.
- The libraries above need to migrate,
  as does EmberData's legacy compatibility layer,
  which forwards `addObserver`/`removeObserver` on records.
- The published types for `@ember/object/observable`,
  `@ember/object/observers`, and the `observer` export from `@ember/object`
  are marked `@deprecated`.

## Exploration

Both changes are already merged on Ember's `next` branch:

- Inline the `Observable` mixin:
  [emberjs/ember.js#20928](https://github.com/emberjs/ember.js/pull/20928)
- Remove the `observer` helper and the inlined `Observable` methods,
  deleting 3,096 lines across 122 files:
  [emberjs/ember.js#20937](https://github.com/emberjs/ember.js/pull/20937)

## How We Teach This

The current guides do not cover observers, so they need no changes.
The work is:

- publish the deprecation guide below
  at [deprecations.emberjs.com](https://deprecations.emberjs.com)
- mark these APIs deprecated in the API docs, linking to the guide
- mention the deprecation in the release blog post

### Deprecation Guide

```js
deprecate(message, false, {
  id: 'deprecate-ember-object-observable',
  until: '8.0.0',
  for: 'ember-source',
  url: 'https://deprecations.emberjs.com/id/deprecate-ember-object-observable',
  since: { available: '7.x', enabled: '7.x' },
});

deprecate(message, false, {
  id: 'deprecate-ember-object-observers',
  until: '8.0.0',
  for: 'ember-source',
  url: 'https://deprecations.emberjs.com/id/deprecate-ember-object-observers',
  since: { available: '7.x', enabled: '7.x' },
});
```

#### What is deprecated

|   | API | status |
| - | --- | ------ |
| 🌐 | the `Observable` mixin (`@ember/object/observable`) | **deprecated** |
| 🌐 | `observer` from `@ember/object` | **deprecated** |
| 🌐 | `addObserver` / `removeObserver` from `@ember/object/observers` | **deprecated** |
| 🌐 | `addObserver` / `removeObserver` as methods on `EmberObject` and its descendants | **deprecated** |

The other `Observable` methods (`get`, `set`, `incrementProperty`, and so on)
are covered by the [`EmberObject` deprecation](https://rfcs.emberjs.com/id/1234-deprecate-ember-object).

#### Migration

<details><summary>Observers that derive state</summary>

```js
// before
import EmberObject, { observer } from '@ember/object';

const Person = EmberObject.extend({
  firstName: null,
  lastName: null,
  fullName: null,

  fullNameDidChange: observer('firstName', 'lastName', function () {
    this.set('fullName', `${this.firstName} ${this.lastName}`);
  }),
});
```

```js
// after
import { tracked } from '@glimmer/tracking';

class Person {
  @tracked firstName;
  @tracked lastName;

  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

The getter computes `fullName` when it is read,
instead of storing a copy each time a dependency changes.

</details>

<details><summary>Observers that perform side effects</summary>

```js
// before
import EmberObject, { observer } from '@ember/object';

const User = EmberObject.extend({
  lastLogin: null,

  lastLoginDidChange: observer('lastLogin', function () {
    this.analytics.track('login', this.lastLogin);
  }),
});

user.set('lastLogin', new Date());
```

```js
// after
import { tracked } from '@glimmer/tracking';

class User {
  @tracked lastLogin;

  login() {
    this.lastLogin = new Date();
    this.analytics.track('login', this.lastLogin);
  }
}

user.login();
```

Run the side effect where the change happens,
not in a callback that watches for it.

</details>

<details><summary>Observers on component arguments</summary>

```js
// before
import Component from '@ember/component';
import { observer } from '@ember/object';
import Chart from 'chart.js';

export default Component.extend({
  tagName: 'canvas',

  didInsertElement() {
    this._super(...arguments);
    this.chart = new Chart(this.element, { type: 'bar', data: this.data });
  },

  dataDidChange: observer('data', function () {
    this.chart.data = this.data;
    this.chart.update();
  }),

  willDestroyElement() {
    this._super(...arguments);
    this.chart.destroy();
  },
});
```

```gjs
// after
import { modifier } from 'ember-modifier';
import Chart from 'chart.js';

const chart = modifier((element, [data]) => {
  let instance = new Chart(element, { type: 'bar', data });

  return () => instance.destroy();
});

<template>
  <canvas {{chart @data}}></canvas>
</template>
```

The modifier re-runs when `@data` changes,
and its returned function runs before each re-run and on teardown.

</details>

<details><summary><code>addObserver</code> / <code>removeObserver</code></summary>

```js
// before
import { addObserver, removeObserver } from '@ember/object/observers';

addObserver(session, 'user', this, 'userDidChange');
removeObserver(session, 'user', this, 'userDidChange');
```

If the observer updates the UI, make `user` a `@tracked` property
and read it in the template or in a getter.
The template re-renders when it changes, with no observer:

```gjs
<template>
  {{#if @session.user}}
    <Dashboard @user={{@session.user}} />
  {{else}}
    <Login />
  {{/if}}
</template>
```

If the observer runs code outside rendering,
such as waiting until a value is set,
there is no efficient replacement yet.
Poll with `requestAnimationFrame` or a timer,
or move the code to the place where the value is set.

</details>

## Drawbacks

Until the follow-up API exists,
libraries that watch state outside rendering
can only poll with `requestAnimationFrame` or timers.

## Alternatives

- Keep the APIs, either by inlining them into `EmberObject`
  so that mixins can be removed, or by doing nothing.
  Rejected because both leave two reactivity systems in place indefinitely,
  and doing nothing also blocks removing mixins.
- Keep the standalone `addObserver`/`removeObserver` functions
  as the low-level integration API.
  Rejected because they expose the legacy eager notification model
  rather than an autotracking primitive.

## Unresolved questions

The design of the low-level API
for consuming tracked state outside the renderer
is left to a follow-up RFC.
