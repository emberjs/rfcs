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

Observers have not been recommended since Octane.
The guides do not teach them, and eslint-plugin-ember's
[`no-observers`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-observers.md)
rule has recommended against them for years.
Removing them shrinks Ember's internals:
[emberjs/ember.js#20937](https://github.com/emberjs/ember.js/pull/20937) alone
deletes 3,096 lines across 122 files.

`Observable` is a mixin, so
[RFC 1116](https://rfcs.emberjs.com/id/1116-deprecate-mixins)
cannot remove mixins until `Observable` is deprecated.

The replacement is autotracking:
`@tracked` properties, native getters,
and modifiers for work that is tied to an element.

## Transition Path

Each deprecated API logs a deprecation when it is called.
The deprecations target `until: '8.0.0'`.

The observer APIs have no direct replacement.
The fix depends on what the observer does:

| Observer used to… | Replace with |
| ----------------- | ------------ |
| derive state from other state | a native getter over `@tracked` properties |
| perform a side effect | an explicit call where the change happens |
| react to classic component arguments | a [modifier](https://guides.emberjs.com/release/components/template-lifecycle-dom-and-modifiers/) |
| watch state outside rendering | no efficient replacement yet (see below) |

Each `Observable` method has a standalone equivalent in `@ember/object`
or a short replacement with `@tracked`.
The [deprecation guide](#deprecation-guide) covers every case.

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

### Ecosystem Impact

- eslint-plugin-ember already ships `no-observers`,
  so no new lint rules are needed.
- The libraries above need to migrate,
  as does EmberData's legacy compatibility layer,
  which forwards `addObserver`/`removeObserver` on records.
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

Every `this.get` and `this.set` call
on a classic component, controller, route, or service
triggers the deprecation.
Each call has a mechanical replacement (see the guide).

Libraries that watch state outside rendering must poll
until the follow-up API ships.

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

Reads are covered: the `ember/no-get` lint rule autofixes `this.get(...)`.
We have not decided whether to ship a codemod for writes,
rewriting `this.set(...)` to `set(this, ...)`.
