---
stage: accepted
start-date: 2026-10-08T00:00:00.000Z
release-date:
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1234 
project-link:
---

<!---
Directions for above:

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
-->

# Deprecate EmberObject 

## Summary

Deprecates `EmberObject` in a way that is initually opt-in, so people can more gradually prepare their codebase for the removal of `EmberObject`. 


## Motivation

`EmberObject` has had heavy use in the early days of Ember (pre-JavaScript having classes), and since classes shipped in 2015, the need for `EmberObject` has greatly diminished. 

Removing `EmberObject` is one of the last steps in coercing codebases to be plain modern JavaScript.

## Transition Path

Unlike previous deprecations, this is targeting Ember 9, and will have a feature flag that removes all behavior related to `EmberObject`. This does require a lot of internal implementation in `ember-source`, but is needed anyway for the removal of `EmberObject`, ultimately.

This will be the first deprecation that users will be able to preview the removal of.

Leading up to v8, the feature flag will be:
  - for existing apps: "off" (someone who hasn't updated their `optional-features.json`):
    - deprecation logged for not having this feature flag "on"
    - EmberObject and all related APIs are still usable (unless the feature flag is "on")
  - for the blueprint: "on" (the setting in `optional-features.json` is set to `true`:
    - new apps cannot throw deprecations, so new apps get the benefits of this feature flag being "on" right away
   
When the feature flag is "on":
  - each API-to-be-removed will throw an error (until we ship the build-time feature-stripping for all the EmberObject and related code)
  - ideally, setting the feature flag to "on" _removes_ all of the implementation for EmberObject, though this is not a blocker for the deprecation's behavior
  - if we aren't able to implement removal in the initial release, we will implement the removal in a future minor release

With the release of v8, and leading up to v9, the feature flag will be "on" by default:
- if users wish, the feature flag can be flipped back off, which brings back the EmberObject behavior along with the deprecation
- when no `optional-features.json` is present, or the `optional-features.json` does not contain the feature flag for this deprecation, the default value is assumed to be "on"

At `ember-source` v9, `EmberObject` is removed fully along with the feature flag.


> [!NOTE]
> This includes `@computed`, as `@computed` is part of the "Ember Object Model" of reactivity.


Internally, implementation would likely be similar to how Mixins were initially deprecated -- copied to an "internal" file, and then the "public" version of `EmberObject` would override `init`, and provide the deprecations. 

On the internal copy of `EmberObject`, we deprecate all the methods (`get`, / `set` / etc), so that the deprecations flow through to other framework classes such as `Route`, `Controller`, `Service`, etc.

## How We Teach This

The guides have not taught `EmberObject` since Octane. The work is:

- publish the deprecation guide below at [deprecations.emberjs.com](https://deprecations.emberjs.com)
- mark `EmberObject`, `@computed`, and the `@ember/object/computed` macros deprecated in the API docs, linking to the guide
- link the [Octane vs Classic cheat sheet](https://guides.emberjs.com/release/upgrading/current-edition/) from the deprecation message; it already has the before/afters

### Deprecation Guide

The deprecation fires once per app boot while the feature flag is off. 

```js
deprecate(message, false, {
  id: 'deprecate-ember-object',
  until: '9.0.0',
  for: 'ember-source',
  url: 'https://deprecations.emberjs.com/id/deprecate-ember-object',
  since: { available: '7.x', enabled: '7.x' }, 
});
```

#### What is deprecated

|   | API | status |
| - | --- | ------ |
| 🌐 | `EmberObject` (default export of `@ember/object`) | **deprecated** |
| 🌐 | `this.get` / `this.set` / `setProperties` / `getProperties` / `incrementProperty` / `toggleProperty` / `notifyPropertyChange` on any class that extends `EmberObject`, including `Route`, `Controller`, `Service` | **deprecated** |
| 🌐 | `init`, `willDestroy`, `destroy`, `isDestroying`, `isDestroyed` as `EmberObject` methods | **deprecated** |
| 🌐 | `reopen` / `reopenClass` | **deprecated** |
| 🌐 | `@computed` and the `@ember/object/computed` macros | **deprecated** |

Related deprecations with their own guides:

- [`.extend()` / `.create()`](https://rfcs.emberjs.com/id/1117-deprecate-classic-classes)
- [Mixins](https://rfcs.emberjs.com/id/1116-deprecate-mixins)
- [observers](https://github.com/emberjs/rfcs/pull/1115)
- [`EmberArray` / `A()`](https://rfcs.emberjs.com/id/1114-deprecate-ember-array)
- [`ObjectProxy` / `ArrayProxy`](https://rfcs.emberjs.com/id/1112-deprecate-proxy)
- [`Evented`](https://rfcs.emberjs.com/id/1111-deprecate-evented-mixin)
- [`@ember/component`](https://rfcs.emberjs.com/id/1216-deprecate-ember-component)

#### Migration

<details><summary>Your own class extends <code>EmberObject</code></summary>

```js
// before
import EmberObject from '@ember/object';

export default class Cart extends EmberObject {
  items = [];

  init() {
    super.init(...arguments);
    this.total = 0;
  }
}

let cart = Cart.create({ currency: 'USD' });
```

```js
// after
export default class Cart {
  items = [];
  total = 0;

  constructor({ currency }) {
    this.currency = currency;
  }
}

let cart = new Cart({ currency: 'USD' });
```

`create()` assigned every key of its argument onto the instance. A constructor receives the same object and assigns what it needs.

</details>

<details><summary><code>this.get</code> / <code>this.set</code></summary>

```js
// before
this.set('count', this.get('count') + 1);
this.setProperties({ name, email });
let { name, email } = this.getProperties('name', 'email');
this.incrementProperty('count');
this.toggleProperty('isOpen');
this.get('user.address.city');
```

```js
// after
this.count = this.count + 1;
Object.assign(this, { name, email });
let { name, email } = this;
this.count++;
this.isOpen = !this.isOpen;
this.user?.address?.city;
```

Assignment only results in a rerender when the property is `@tracked`. [ember-tracked-properties-codemod](https://github.com/ember-codemods/ember-tracked-properties-codemod) adds `@tracked` to properties that `set` wrote to. The `ember/no-get` lint rule autofixes the reads.

`notifyPropertyChange` has no replacement. With `@tracked`, the write is the notification.

</details>

<details><summary><code>@computed</code></summary>

```js
// before
import { computed } from '@ember/object';
import { alias, filterBy, sort } from '@ember/object/computed';

export default class Cart extends EmberObject {
  @computed('items.@each.price')
  get total() {
    return this.items.reduce((sum, item) => sum + item.price, 0);
  }

  @alias('user.name') owner;
  @filterBy('items', 'isGift', true) gifts;
  @sort('items', 'sortKeys') sorted;
}
```

```js
// after
import { cached } from '@glimmer/tracking';

export default class Cart {
  @cached
  get total() {
    return this.items.reduce((sum, item) => sum + item.price, 0);
  }

  get owner() { return this.user.name; }
  get gifts() { return this.items.filter((item) => item.isGift); }
  get sorted() { return this.items.toSorted(byKeys(this.sortKeys)); }
}
```

Dependent keys go away. A getter re-runs when any `@tracked` value it read changes. Use `@cached` only when the getter is expensive.

</details>

<details><summary><code>willDestroy</code> / <code>destroy()</code></summary>

```js
// before
export default class Poller extends EmberObject {
  init() {
    super.init(...arguments);
    this.timer = setInterval(this.tick, 1000);
  }

  willDestroy() {
    clearInterval(this.timer);
    super.willDestroy(...arguments);
  }
}

poller.destroy();
```

```js
// after
import { registerDestructor, destroy } from '@ember/destroyable';

export default class Poller {
  constructor() {
    this.timer = setInterval(this.tick, 1000);
    registerDestructor(this, () => clearInterval(this.timer));
  }
}

destroy(poller);
```

`isDestroying` and `isDestroyed` are also exported from `@ember/destroyable`. `Route`, `Controller`, and `Service` keep `willDestroy`.

</details>

<details><summary><code>Route</code>, <code>Controller</code>, <code>Service</code></summary>

Only the `EmberObject` methods on these classes are deprecated:

```js
// before
import Service from '@ember/service';
import { computed } from '@ember/object';

export default class Session extends Service {
  init() {
    super.init(...arguments);
    this.set('user', null);
  }

  @computed('user')
  get isLoggedIn() {
    return Boolean(this.get('user'));
  }
}
```

```js
// after
import Service from '@ember/service';
import { tracked } from '@glimmer/tracking';

export default class Session extends Service {
  @tracked user = null;

  get isLoggedIn() {
    return Boolean(this.user);
  }
}
```

</details>

<details><summary><code>reopen</code> / <code>reopenClass</code></summary>

```js
// before
Cart.reopen({ currency: 'USD' });
Cart.reopenClass({ fromJSON(json) { /* ... */ } });
```

```js
// after
export default class Cart {
  currency = 'USD';

  static fromJSON(json) { /* ... */ }
}
```

If the class is not yours, you may use the Presenter pattern for wrapping/enriching the source data. Addons that expected consumers to `reopen` their classes need to expose a configuration API instead.

</details>

## Drawbacks

keeping EmberObject is a drawback, because of the dozens of KB that come along with it.

all codebases with old code probably have some usage of EmberObject remaining, so they need to migrate.

## Alternatives

- do nothing

## Unresolved questions

n/a
