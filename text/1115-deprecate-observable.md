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

Deprecate observers and the `Observable` mixin:
the `observer` helper, the `addObserver` and `removeObserver` functions,
and the methods that `Observable` adds to `EmberObject` and its descendants.
The standalone `get`, `set`, `getProperties`, `setProperties`,
and `notifyPropertyChange` functions from `@ember/object` are not deprecated
by this RFC.

## Motivation

Observers have been informally discouraged
since the introduction of autotracking.
The guides do not teach them, and eslint-plugin-ember's
[`no-observers`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-observers.md)
rule has recommended against them for years.
Removing support for them will simplify Ember's internals.
(To remove observer support from Ember's internals completely,
we will need to deprecate other functionality
in addition to what is included here.)

Additionally, Ember has not recommended mixins for several years.
Because `Observable` is a mixin,
it must be deprecated before we can remove mixins entirely.

The replacement is autotracking:
`@tracked` properties, native getters,
and modifiers for work that is tied to an element.

## Transition Path

Each deprecated API logs a deprecation when it is called.
The deprecations target `until: '8.0.0'`.

The `observer` helper does not work with native class syntax
and, as such, should not be relied upon.
It will not be replaced,
and neither will the standalone `addObserver` and `removeObserver` functions.
The transition path depends on what the observer was doing.

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

The `Observable` methods either have a standalone equivalent in `@ember/object`
or a short replacement with `@tracked`.
The deprecation guide below covers each one.

### Integration Outside Rendering

Some libraries use observers to learn when state changes outside rendering.
ember-concurrency's [`waitForProperty`](https://github.com/machty/ember-concurrency/blob/0ffd122ed65f68970e0535ca2e54779c0140cac7/packages/ember-concurrency/src/-private/wait-for.js#L97-L119)
re-checks its predicate each time the observed property changes.
ember-animated uses observers in its task
[`.observes()`](https://github.com/ember-animation/ember-animated/blob/f0a53fb6c947f9018129dc1b97e21339397612ed/addon/src/-private/ember-scheduler.ts#L76-L85) modifier
and in [`<AnimatedEach>`](https://github.com/ember-animation/ember-animated/blob/f0a53fb6c947f9018129dc1b97e21339397612ed/addon/src/components/animated-each.gts#L222-L241),
and EmberData's [Inspector data adapter](https://github.com/emberjs/data/blob/a8ac10f9b2f8e2280b44fed40dedf5fb872cd5f4/packages/debug/src/data-adapter.ts#L739-L758)
observes records to push updates to the Ember Inspector.
Today the only workarounds are polling with `requestAnimationFrame` or timers.

Ember should expose a deliberately low-level API
for consuming tracked state outside the renderer,
probably building on `createCache` / `getValue`
with a way to be notified of invalidation.
Designing that API is left to a follow-up RFC.

### Ecosystem Impact

- eslint-plugin-ember already ships `no-observers`,
  so no new lint rules are needed.
  The rule's docs can link to the deprecation guide.
- Addons that use observers will need to migrate,
  including ember-concurrency, ember-animated,
  and EmberData's Inspector adapter and legacy compatibility layer.
  The `until: '8.0.0'` window gives them time to do so.
- The published types for `@ember/object/observable`,
  `@ember/object/observers`, and the `observer` export from `@ember/object`
  will be marked `@deprecated` and removed along with the APIs.

## Exploration

To validate this deprecation, we removed this functionality on Ember's `next`
branch in the following merged PRs:

- Inline the `Observable` mixin:
  [emberjs/ember.js#20928](https://github.com/emberjs/ember.js/pull/20928)
- Remove the `observer` helper and the inlined `Observable` methods:
  [emberjs/ember.js#20937](https://github.com/emberjs/ember.js/pull/20937)

## How We Teach This

The current guides do not cover observers, so they need no changes.
The work is:

- publish the deprecation guide below
  at [deprecations.emberjs.com](https://deprecations.emberjs.com)
- mark the deprecated APIs as deprecated in the API docs, linking to the guide
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
| 🌐 | `get` / `set` / `getProperties` / `setProperties` / `notifyPropertyChange` as methods on `EmberObject` and its descendants | **deprecated** |
| 🌐 | `incrementProperty` / `decrementProperty` / `toggleProperty` / `cacheFor` as methods on `EmberObject` and its descendants | **deprecated** |
| 🌐 | `addObserver` / `removeObserver` as methods on `EmberObject` and its descendants | **deprecated** |
| 🌐 | `observer` from `@ember/object` | **deprecated** |
| 🌐 | `addObserver` / `removeObserver` from `@ember/object/observers` | **deprecated** |
| 🌐 | `get` / `set` / `getProperties` / `setProperties` / `notifyPropertyChange` from `@ember/object` | stays |

Related deprecations with their own guides:

- [`EmberObject`](https://rfcs.emberjs.com/id/1234-deprecate-ember-object)
- [Mixins](https://rfcs.emberjs.com/id/1116-deprecate-mixins)
- [`.extend()` / `.create()`](https://rfcs.emberjs.com/id/1117-deprecate-classic-classes)
- [`ObjectProxy` / `ArrayProxy`](https://rfcs.emberjs.com/id/1112-deprecate-proxy)

#### Migration

<details><summary><code>this.get</code> / <code>this.set</code> and the other property methods</summary>

```js
// before
person.set('firstName', 'John');
let name = person.get('firstName');
person.setProperties({ firstName: 'John', lastName: 'Doe' });
let { firstName, lastName } = person.getProperties('firstName', 'lastName');
person.notifyPropertyChange('firstName');
```

Code that still uses `@computed` or Ember's proxies
can switch to the standalone functions:

```js
// after: @computed or proxies
import { get, set, getProperties, setProperties, notifyPropertyChange } from '@ember/object';

set(person, 'firstName', 'John');
let name = get(person, 'firstName');
setProperties(person, { firstName: 'John', lastName: 'Doe' });
let { firstName, lastName } = getProperties(person, 'firstName', 'lastName');
notifyPropertyChange(person, 'firstName');
```

Once the properties are `@tracked`,
plain property access and assignment work:

```js
// after: @tracked
import { tracked } from '@glimmer/tracking';

class Person {
  @tracked firstName;
  @tracked lastName;
}

person.firstName = 'John';
let name = person.firstName;
Object.assign(person, { firstName: 'John', lastName: 'Doe' });
let { firstName, lastName } = person;
```

`notifyPropertyChange` has no equivalent with `@tracked`.
The write is the notification.

</details>

<details><summary><code>incrementProperty</code> / <code>decrementProperty</code> / <code>toggleProperty</code></summary>

```js
// before
this.incrementProperty('count');
this.decrementProperty('count');
this.toggleProperty('flag');
```

```js
// after: @computed or proxies
import { get, set } from '@ember/object';

set(this, 'count', get(this, 'count') + 1);
set(this, 'count', get(this, 'count') - 1);
set(this, 'flag', !get(this, 'flag'));
```

```js
// after: @tracked
this.count += 1;
this.count -= 1;
this.flag = !this.flag;
```

</details>

<details><summary><code>cacheFor</code></summary>

```js
// before
import { computed } from '@ember/object';

class Cart extends EmberObject {
  @computed('items.@each.price')
  get total() {
    return this.items.reduce((sum, item) => sum + item.price, 0);
  }
}

let total = cart.cacheFor('total');
```

```js
// after
import { cached } from '@glimmer/tracking';

class Cart {
  @cached
  get total() {
    return this.items.reduce((sum, item) => sum + item.price, 0);
  }
}

let total = cart.total;
```

`@cached` memoizes the getter until a `@tracked` value it read changes.
There is no replacement for peeking at a cached value without computing it.

</details>

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

Classic code calls `this.get` and `this.set` on `EmberObject` subclasses
(components, controllers, routes, and services) very often, so many apps will
see a large number of deprecation warnings. Each call has a mechanical
replacement, but the churn is real. We do not want to support these APIs
long-term, so we accept that cost.

Non-rendering integration with the reactivity system has no efficient
replacement until the low-level API described above exists.

## Alternatives

- Inline this functionality into `EmberObject` so that we can remove mixins,
  but keep the APIs.
  Rejected because it leaves two reactivity systems in place.
- Keep the standalone `addObserver`/`removeObserver` functions as the
  low-level integration API. Rejected because they expose the legacy eager
  notification model rather than an autotracking primitive.
- Do nothing. Rejected because it blocks removing mixins and means
  maintaining two reactivity systems indefinitely.

## Unresolved questions

The design of the low-level API for consuming tracked state outside the
renderer is left to a follow-up RFC.

We have not decided whether to ship a codemod that rewrites `this.get(...)`
and `this.set(...)` to the standalone `get(this, ...)` and `set(this, ...)`.
