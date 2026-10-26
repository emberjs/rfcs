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

Deprecate the observer APIs: the `observer` helper, `addObserver` and `removeObserver` (both the `@ember/object/observers` functions and the `EmberObject` methods), and the `Observable` mixin.
The other methods that `Observable` adds to `EmberObject`, such as `get` and `set`, are deprecated along with `EmberObject` by [RFC 1234](https://rfcs.emberjs.com/id/1234-deprecate-ember-object).

## Motivation

Observers have not been recommended since Octane replaced them with autotracking, and eslint-plugin-ember's [`no-observers`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-observers.md) rule has flagged them for years.
Removing them shrinks Ember's internals (see [Exploration](#exploration)).

`Observable` is a mixin, so [RFC 1116](https://rfcs.emberjs.com/id/1116-deprecate-mixins) cannot remove mixins until `Observable` is deprecated.

## Transition Path

Each deprecated API logs a deprecation when it is called, or, for `Observable`, when the mixin is applied.
The deprecations target `until: '8.0.0'`.

No API directly replaces observers.
The fix depends on what the observer does, and the [deprecation guide](#deprecation-guide) has an example of each:

| Observer used to… | Replace with |
| ----------------- | ------------ |
| derive state from other state | a native getter over `@tracked` properties |
| perform a side effect | an explicit call where the change happens |
| react to classic component arguments | a function called from the template |

### Ecosystem Impact

- eslint-plugin-ember already ships `no-observers`, so no new lint rules are needed.
- These libraries use observers and need to migrate:
  - ember-concurrency's [`waitForProperty`](https://github.com/machty/ember-concurrency/blob/0ffd122ed65f68970e0535ca2e54779c0140cac7/packages/ember-concurrency/src/-private/wait-for.js#L97-L119)
  - ember-animated's task [`.observes()`](https://github.com/ember-animation/ember-animated/blob/f0a53fb6c947f9018129dc1b97e21339397612ed/addon/src/-private/ember-scheduler.ts#L76-L85) modifier and [`<AnimatedEach>`](https://github.com/ember-animation/ember-animated/blob/f0a53fb6c947f9018129dc1b97e21339397612ed/addon/src/components/animated-each.gts#L222-L241)
  - EmberData's [Inspector data adapter](https://github.com/emberjs/data/blob/a8ac10f9b2f8e2280b44fed40dedf5fb872cd5f4/packages/debug/src/data-adapter.ts#L739-L758) and its legacy compatibility layer, which forwards `addObserver`/`removeObserver` on records
- The published types for `@ember/object/observable`, `@ember/object/observers`, and the `observer` export from `@ember/object` are marked `@deprecated`.

## Exploration

Both changes are already merged on Ember's `next` branch:

- Inline the `Observable` mixin: [emberjs/ember.js#20928](https://github.com/emberjs/ember.js/pull/20928)
- Remove the `observer` helper and the inlined `Observable` methods, deleting 3,096 lines across 122 files: [emberjs/ember.js#20937](https://github.com/emberjs/ember.js/pull/20937)

## How We Teach This

The current guides do not cover observers, so they need no changes.
The work is:

- publish the deprecation guide below at [deprecations.emberjs.com](https://deprecations.emberjs.com)
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

The other `Observable` methods (`get`, `set`, `incrementProperty`, and so on) are covered by the [`EmberObject` deprecation](https://rfcs.emberjs.com/id/1234-deprecate-ember-object).

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

The getter computes `fullName` when it is read, instead of storing a copy each time a dependency changes.

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

Run the side effect where the change happens, not in a callback that watches for it.

</details>

<details><summary>Observers on component arguments</summary>

```js
// before
import Component from '@ember/component';
import { observer } from '@ember/object';

export default Component.extend({
  titleDidChange: observer('title', function () {
    document.title = this.title;
  }),
});
```

```gjs
// after
function setTitle(title) {
  document.title = title;
}

<template>
  {{setTitle @title}}
</template>
```

The template calls `setTitle` again whenever `@title` changes.
If the function sets tracked state, wrap it in [`effect`](https://github.com/universal-ember/reactiveweb/blob/main/reactiveweb/src/effect.ts) from reactiveweb so that it runs after rendering.

</details>

<details><summary><code>addObserver</code> / <code>removeObserver</code></summary>

```js
// before
import { addObserver, removeObserver } from '@ember/object/observers';

addObserver(session, 'user', this, 'userDidChange');
removeObserver(session, 'user', this, 'userDidChange');
```

If the observer updates the UI, make `user` a `@tracked` property and read it in the template or in a getter.
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

If the observer runs code when the value changes, call that code where the value is set.
You can also call it from the template that reads the value, as in the previous section.

</details>

## Drawbacks

Code that reacted to a change from anywhere must now run either where the change happens or from a template that reads the value.
Libraries that relied on observers for this, such as ember-concurrency's `waitForProperty`, need a new design.

## Alternatives

- Keep the APIs, either by inlining them into `EmberObject` so that mixins can be removed, or by doing nothing.
  Rejected because both leave two reactivity systems in place indefinitely, and doing nothing also blocks removing mixins.

## Unresolved questions

None.
