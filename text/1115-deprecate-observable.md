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

Deprecate the `observer` helper from `@ember/object` and the `addObserver` and `removeObserver` functions from `@ember/object/observers`.
The `Observable` mixin and the methods it adds to `EmberObject` are already deprecated by [RFC 1116](https://rfcs.emberjs.com/id/1116-deprecate-mixins) and [RFC 1234](https://rfcs.emberjs.com/id/1234-deprecate-ember-object).

## Motivation

Observers have not been recommended since Octane replaced them with autotracking, and eslint-plugin-ember's [`no-observers`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-observers.md) rule has flagged them for years.
Removing them shrinks Ember's internals (see [Exploration](#exploration)).

## Transition Path

Calling a deprecated API logs a deprecation that targets `until: '8.0.0'`.

No API directly replaces observers.
The fix depends on what the observer does, and the [deprecation guide](#deprecation-guide) has an example of each:

| Observer used to… | Replace with |
| ----------------- | ------------ |
| derive state from other state | a native getter over `@tracked` properties |
| perform a side effect | an explicit call where the change happens |
| react to classic component arguments | a function called from the template |

### Ecosystem Impact

- eslint-plugin-ember already ships `no-observers`, so no new lint rules are needed.
- Several libraries use observers and need to migrate (see [Library Usage](#library-usage)).
- The published types for `@ember/object/observers` and the `observer` export from `@ember/object` are marked `@deprecated`.

## How We Teach This

The current guides do not cover observers, so they need no changes.
The work is:

- publish the deprecation guide below at [deprecations.emberjs.com](https://deprecations.emberjs.com)
- mark these APIs deprecated in the API docs, linking to the guide
- mention the deprecation in the release blog post

### Deprecation Guide

```js
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
| 🌐 | `observer` from `@ember/object` | **deprecated** |
| 🌐 | `addObserver` / `removeObserver` from `@ember/object/observers` | **deprecated** |

The `Observable` mixin and its methods, including `this.addObserver`, are covered by the [Mixins](https://rfcs.emberjs.com/id/1116-deprecate-mixins) and [`EmberObject`](https://rfcs.emberjs.com/id/1234-deprecate-ember-object) deprecations.

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
Alternatively, wrap it in [`effect`](https://github.com/universal-ember/reactiveweb/blob/main/reactiveweb/src/effect.ts) from reactiveweb so that it runs after rendering. 

> [!NOTE]
> Setting tracked state at all in an effect-like primitive is dangerous and can cause an infinite loop -- even if it's a slower infinite loop.

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
Libraries that relied on observers for this need a new design.

## Alternatives

- Do nothing.
  This keeps two reactivity systems in place indefinitely.

## Unresolved questions

None.

## Appendix

### Exploration

[emberjs/ember.js#20937](https://github.com/emberjs/ember.js/pull/20937), merged on Ember's `next` branch, removes the `observer` helper along with the inlined `Observable` methods, deleting 3,096 lines across 122 files.

### Library Usage

These libraries use observers and need to migrate:

- ember-concurrency's [`waitForProperty`](https://github.com/machty/ember-concurrency/blob/0ffd122ed65f68970e0535ca2e54779c0140cac7/packages/ember-concurrency/src/-private/wait-for.js#L97-L119)
- ember-animated's task [`.observes()`](https://github.com/ember-animation/ember-animated/blob/f0a53fb6c947f9018129dc1b97e21339397612ed/addon/src/-private/ember-scheduler.ts#L76-L85) modifier and [`<AnimatedEach>`](https://github.com/ember-animation/ember-animated/blob/f0a53fb6c947f9018129dc1b97e21339397612ed/addon/src/components/animated-each.gts#L222-L241)
- EmberData's [Inspector data adapter](https://github.com/emberjs/data/blob/a8ac10f9b2f8e2280b44fed40dedf5fb872cd5f4/packages/debug/src/data-adapter.ts#L739-L758) and its legacy compatibility layer, which forwards `addObserver`/`removeObserver` on records
