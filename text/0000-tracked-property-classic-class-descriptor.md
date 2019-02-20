- Start Date: 2018-02-07
- Relevant Team(s): Ember.js, Learning
- RFC PR: https://github.com/emberjs/rfcs/pull/442
- Tracking: (leave this empty)

# Add `descriptor()`

## Summary

Add a `descriptor` decorator for classic classes only which allows users to
define native getters and setters (and other properties):

```js
import EmberObject, { descriptor } from '@ember/object';
import { tracked } from '@glimmer/tracking';

const Person = EmberObject.extend({
  firstName: tracked({ value: 'Tom' }),
  lastName: tracked({ value: 'Tom' }),

  fullName: descriptor({
    get() {
      return `${this.firstName} ${this.lastName}`;
    },
  }),
});
```

## Motivation

The Tracked Properties RFC specified that classic classes would be supported,
and it would be possible to do everything that we can do with native syntax
using classic syntax. Now that native decorator support requires us to maintain
classic classes for the forseeable future, this is doubly important.
Additionally, it will be easier for the Learning team to be able to have classic
and native examples for each piece of functionality side-by-side, without having
to constantly explain caveats.

In the original RFC, classic classes were demonstrated using `tracked` to
provide a getter:

```js
import EmberObject from '@ember/object';
import { tracked } from '@glimmer/tracking';

const Person = EmberObject.extend({
  firstName: tracked({ value: 'Tom' }),
  lastName: tracked({ value: 'Tom' }),

  fullName: tracked({
    get() {
      return `${this.firstName} ${this.lastName}`;
    },
  }),
});
```

This is confusing because it implies that the getter itself is tracked.
Additionally, if we ever want to _make_ getters/setters trackable in the future,
this would be problematic since it would have already implicitly locked in the
behavior of `@tracked` being applied to getters/setters.

`descriptor` will also fill in some minor gaps left by deprecating `volatile`,
which was occasionally used to simulate a standard ES5 getter on objects.

## Detailed design

`descriptor` will receive a standard ES5+ property descriptor, and apply it to
the prototype of the class:

```ts
function descriptor(desc: PropertyDescriptor): PropertyDecorator;
```

Any valid property descriptor will be allowed, following the rules of
[`Object.defineProperty`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty).

## How we teach this

`descriptor` should be taught during the classic class introduction in the
guides, and should primarily be taught as a way to add _getters_ and _setters_
to classic classes. Other uses are covered in other ways, so they shouldn't be
highlighted.

All examples including tracked properties in the guides which have a classic
class equivalent will use `descriptor` instead of native getters, demonstrating
how it works by example.

## Drawbacks

Adds more functionality to classic classes, which may not be the direction we
want to go in the long run. However, this is important for creating a base of
stability as we transition to native classes, and in general will not be
highlighted as the preferred/happy path as we move forward.

## Alternatives

- Not provide this functionality, and accept the gap. This would be painful for
  the learning team as we begin writing the Octane guides, since a simple 1-1
  translation is actually the _easiest_ way to both completely support classic
  classes _and_ simultaneously de-emphasize them. Constantly calling out random
  differences means the team has to devote _more_ time to classic classes, which
  counterintuitively means we end up spending more time calling out that they -
  exist and are supported.

- Lock down `descriptor` to just allow `get` and `set`. This leads to an awkward
  naming problem (do we call it `accessor`?).
