---
Stage: Accepted
Start Date: 2022-03-29
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember CLI, Learning
RFC PR: https://github.com/emberjs/rfcs/pull/812
---

<!--- 
Directions for above: 

Stage: Leave as is
Start Date: Fill in with today's date, YYYY-MM-DD
Release Date: Leave as is
Release Versions: Leave as is
Relevant Team(s): Fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies
RFC PR: Fill this in with the URL for the Proposal RFC PR
-->

# Add tracked-built-ins

## Summary

This RFC proposes adding [tracked-built-ins](https://github.com/tracked-tools/tracked-built-ins)
to the blueprints that back `ember new` and `ember addon`.

## Motivation

Ember had historically shipped `EmberArray` and `EmberObject`
as well as `Map` and `Set` implementations to make it easy
to work with those types in a reactive way.
`tracked-built-ins` does the same for the auto-tracking era.

Autotracking (Ember's reactivity model) is one of the key features of
[Ember Octane](https://emberjs.com/editions/octane/). `tracked-built-ins` fills the gap
in the reactivity mental model we have today when it comes to tracking updates in data structures.

Today, `@tracked` decorator may be used to track updates of the primitive values:

```js
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { action } from '@ember/object';

export default class CounterComponent extends Component {
  @tracked count = 0;

  @action increment() {
    this.count = this.count + 1;
  }

  @action decrement() {
    this.count = this.count - 1;
  }
}
```

However, there are cases when there are unknown, dynamic, or potentially infinite numbers
of decoratable keys. In such cases we can use `Object`, `Map` or another data structure
that fits most in the particular use case. The initial pattern proposed for Ember Octane
in such scenarios was to re-set the tracked property:

```js
class TrackedMap {
  @tracked map = new Map();

  updateMapValue(key, value) {
    this.map.set(key, value);
    this.map = this.map;
  }
}
```

This pattern is problematic however, because it's not obvious what the
purpose of re-setting the property is. A developer unfamiliar with Ember may
mistakenly believe this was an error, and remove this statement, causing
bugs. This pattern is not easy to search on the Internet, which makes it more
difficult to learn and to teach. Finally, it's fairly easy this way for the
state to get out of sync with the rest of the system if a developer forgets
to add the extra re-set after mutating the value.

Over time, it's become clear that this pattern is actually an anti-pattern
for these reasons, and something we should begin to discourage in general.

[RFC #669 "Tracked Storage Primitives"][rfc-0669] introduced low level primitives
that allow to implement tracked versions of JavaScript's built-in classes
in an addon like [tracked-built-ins](https://github.com/tracked-tools/tracked-built-ins):

```ts
import {
  TrackedObject,
  TrackedArray,
  TrackedMap,
  TrackedSet,
  TrackedWeakMap,
  TrackedWeakSet,
} from 'tracked-built-ins';

class Foo {
  @tracked value = 123;

  obj = new TrackedObject<{[string: number]}>({ foo: 123 });
  arr = new TrackedArray<string, number>(['foo', 123]);
  map = new TrackedMap<string, number>(['foo', 123]);
  set = new TrackedSet<string, number>(['foo', 123]);
  weakMap = new TrackedWeakMap<{[object: number]}>([[{}, 456]]);
  weakSet = new TrackedWeakSet<string, number>(['foo', 123]);
}
```

or using `tracked()` decorator:

```ts
import { tracked } from 'tracked-built-ins';

class Foo {
  @tracked value = 123;

  obj = tracked<{[string: number]}>({ foo: 123 });
  arr = tracked<string, number>(['foo', 123]);
  map = tracked<string, number>(new Map(['foo', 123]));
  set = tracked<string, number>(new Set(['foo', 123]));
  weakMap = tracked<{[string: number]}>(new WeakMap([[{}, 456]]));
  weakSet = tracked<string, number>(new WeakSet(['foo', 123]));
}
```

## Detailed design

The necessary changes to `ember-cli` are relatively small since we only need
to add the dependency to the `app` blueprint, and the `addon` blueprint will
inherit it automatically.

This has the advantage (over including it as an implicit dependency), that
apps and addons that don't want to use it for some reason can opt out by
removing the dependency from their `package.json` file.

## How we teach this

The following guide should be added to the Autotracking In-Depth guide in the official guides, and the
sections on [POJOs][guides-pojos] and [Array][guides-arrays] should be removed.

### Tracked data structures

Generally, you should try to create classes with their tracked properties enumerated
and decorated with `@tracked`, instead of relying on dynamically created objects.
In some cases however, there could be a prohibitive number of possible properties,
or there could be no way to know them in advance.
In this case, you should use tracked versions of JavaScript's built-in
Plain Old JavaScript Object (POJO), array or keyed collection (Maps, Sets, WeakMaps, WeakSets)
provided by `tracked-built-ins` package.

```js
import {
  TrackedObject,
  TrackedArray,
  TrackedMap,
  TrackedSet,
  TrackedWeakMap,
  TrackedWeakSet,
} from 'tracked-built-ins';
```

For example, we could make a scoreboard component that keeps score for an
arbitrary number of players, and keep track of the score for each player using
tracked version of JavaScript [Map][global-objects-map].

**Note:** In this and the following examples, I am using `<template>` from
[RFC #779][rfc-0779].

```js
// app/components/scoreboard.js
import Component from '@glimmer/component';
import { action } from '@ember/object';
import { TrackedMap } from 'tracked-built-ins';

const PLAYERS = [
  { name: 'Zoey' },
  { name: 'Tomster' },
];

export default class Scoreboard extends Component {
  // Create a map from player -> score, with each player's score starting at 0
  scores = new TrackedMap(PLAYERS.map(p => [p, 0]));

  @action
  incrementScore(player) {
    let currentScore = this.scores.get(player);

    this.scores.set(player, currentScore + 1);
  }

  <template>
    {{#each-in this.scores |player score|}}
      <div>
        {{player.name}}: {{score}}
    
        <button type="button" {{on "click" (fn this.incrementScore player)}}>
          +
        </button>
      </div>
    {{/each-in}}
  </template>
}
```

Let's take a look into another example involving a shopping cart, where we keep track
of the items added to cart using tracked version of JavaScript's [Array][global-objects-array].

```js
// app/components/scoreboard.js
import Component from '@glimmer/component';
import { action } from '@ember/object';
import { TrackedArray } from 'tracked-built-ins';

export default class Cart extends Component {
  items = new TrackedArray([
    { productName: 'Jeans' },
    { productName: 'T-shirt' },
  ]);

  @action
  addPants(item) {
    this.items.push(item);
  }

  <template>
    {{#each this.items as |item|}}
      <div>
        {{item.productName}}
      </div>
    {{/each}}
  
    <button type="button" {{on "click" (fn this.addPants @pants)}}>
      Add Pants
    </button>
  </template>
}
```

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

- `tracked-built-ins` can be added only to the `app` blueprint and not to the `addon` blueprint.

- Do nothing and recommend using re-setting the property to track data structure changes.

  The tradeoffs of this approach described in "Motivation" section of this RFC.

- Don't add `tracked-built-ins` to the blueprint and only update Ember Guides.

  Ember.js is advertised as "batteries included" framework with complete out-of-the-box experience.
  Asking users to install additional dependencies to achieve relatively common problem contradicts
  with that paradigm.

## Unresolved questions

- What, if any, substantive changes to the APIs presented by `tracked-built-ins` should we make?

- Should we introduce these at something like `@glimmer/tracking` instead?

  ```js
  import { tracked, TrackedMap, TrackedSet /* etc. */ } from '@glimmer/tracking';
  ```

- Alternatively, should they have a dedicated module for each type?

  ```js
  import TrackedMap from '@glimmer/tracking/map';
  import TrackedSet from '@glimmer/tracking/set';
  // etc.
  ```

[rfc-0669]: https://emberjs.github.io/rfcs/0669-tracked-storage-primitive.html
[rfc-0779]: https://emberjs.github.io/rfcs/0779-first-class-component-templates.html
[guides-pojos]: https://guides.emberjs.com/release/in-depth-topics/autotracking-in-depth/#toc_plain-old-javascript-objects-pojos
[guides-arrays]: https://guides.emberjs.com/release/in-depth-topics/autotracking-in-depth/#toc_arrays
[global-objects-map]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map
[global-objects-array]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array