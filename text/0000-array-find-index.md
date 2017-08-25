- Start Date: 2017-08-25
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

This RFC aims to add the [new ES6 Array method `findIndex`][mdn-findIndex]
and a complimentary `findIndexBy` method to the
[`Ember.Array`][ember-array] class.

# Motivation

`findIndex` is one of the few methods missing on `Ember.Array` for
full feature parity with native ES6 Arrays. It is similar to the [`find`][ember-array-find] method in that it accepts a `callback`
function and optional `target` object (as the context for `this`).
Instead of returning the first matching element itself, if any,
`findIndex` returns the index of that element, otherwise `-1`.

For convenience and symmetry with [`find`][ember-array-find] and
[`findBy`][ember-array-findBy], this RFC also proposes that an
accompanying `findIndexBy` method be added.

Why this is useful should be obvious. For current alternatives, refer to the
*Alternatives* section.

# Detailed design

#### `Enumerable` vs. `Array`

The main difference between [`Ember.Enumerable`][ember-enumerable] and
[`Ember.Array`][ember-array] is that the latter offers explicit index
based access to elements.

> Unlike `Ember.Enumerable`, this mixin defines methods specifically
> for collections that provide index-ordered access to their contents.
> When you are designing code that needs to accept any kind of
> Array-like object, you should use these methods instead of Array
> primitives because these will properly notify observers of changes
> to the array.

Since `Enumerable`s are not intended to be used for direct index based
access, but only enumeration / iteration, it does not make a lot of
sense to find the index of an element in an `Enumerable`. Therefore
`findIndex` should be only available on `Ember.Array` and not
`Ember.Enumerable`.

While it probably wouldn't *hurt* to add it to `Enumerable` as well,
I would advise against it in this initial RFC, since it would force us
to keep support for it until Ember 3.0.

If later at some point the core team decides that adding it to
`Enumerable` as well is advantageous and does not pose a problem,
moving the method to `Enumerable` is fully backwards compatible.

#### `findIndex`

The current implementation of [`find`][code-find] is already pretty close to
what `findIndex` would need to look like. The only difference is that instead of
returning the actual found element, `findIndex` would need to return the index.

A straight copy of `Enumerable#find` with the required edit would look like this:

```js
// ember-runtime/lib/mixins/array.js
// Note: popCtx and pushCtx are not available in this file. This is
//       purely for illustrative purposes.
find(callback, target) {
  assert('Array#findIndex expects a function as first argument.', typeof callback === 'function');

  let len = get(this, 'length');

  if (target === undefined) {
    target = null;
  }

  let context = popCtx();
  let found = false;
  let last = null;
  let next, ret;

  for (let idx = 0; idx < len && !found; idx++) {
    next = this.nextObject(idx, last, context);

    found = callback.call(target, next, idx, this);
    if (found) {
      // ret = next;
      ret = idx;
    }

    last = next;
  }

  next = last = null;
  context = pushCtx(context);

  return ret;
}
```

In order to stay DRY, this common logic should be factored out into a private,
shared utility method that returns a double tuple or object:

```js
// ember-runtime/lib/mixins/enumerable.js
findElementAndIndex(callback, target) {
  assert('Enumerable#findElementAndIndex expects a function as first argument.', typeof callback === 'function');

  let len = get(this, 'length');

  if (target === undefined) {
    target = null;
  }

  let context = popCtx();
  let found = false;
  let last = null;
  let next, ret;

  for (let idx = 0; idx < len && !found; idx++) {
    next = this.nextObject(idx, last, context);

    found = callback.call(target, next, idx, this);
    if (found) {
      ret = [next, idx];
    }

    last = next;
  }

  next = last = null;
  context = pushCtx(context);

  return ret || [null, -1];
}
```

This common method may then be used in `find` and `findIndex`:

```js
// ember-runtime/lib/mixins/enumerable.js
find(callback, target) {
  assert('Enumerable#find expects a function as first argument.', typeof callback === 'function');

  return this.findElementAndIndex(callback, target)[0];
}
```

```js
// ember-runtime/lib/mixins/array.js
findIndex(callback, target) {
  assert('Array#findIndex expects a function as first argument.', typeof callback === 'function');

  return this.findElementAndIndex(callback, target)[1];
}
```

#### `findIndexBy`

This method will be implemented analogous to [`findBy`][code-findBy].
However, since `findBy` accesses the utility function
[`iter`][code-iter] it would either need to be exported or its
functionaliy copied.

```js
// ember-runtime/lib/mixins/array.js
import { iter } from './enumerable';

findIndexBy(key, value) {
  return this.findIndex(iter.apply(this, arguments));
}
```

# How We Teach This

##### What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation of existing Ember patterns, or as a wholly new one?

Proficient users of JavaScript would expect `findIndex` to be available on
`Ember.Array`, since it is a standardized ES6 method. Even if you have never
heard of it, the name gives you a pretty good idea of what this method is
supposed to do.

`findIndexBy` complements `findIndex` in the same way that `findBy` complements
`find`. It exactly fits the existing mental model of `{something}By` Array
methods:

- `filter` / `filterBy`
- `find` / `findBy`
- `map` / `mapBy`
- `reject` / `rejectBy`
- `sort` / `sortBy`
- `uniq` / `uniqBy`

##### Would the acceptance of this proposal mean the Ember guides must be re-organized or altered?

Apart from adding the usual YUIDoc inline documentation, no.

##### Does it change how Ember is taught to new users at any level?

No.

##### How should this feature be introduced and taught to existing Ember users?

As usual, mentioning the PR in the changelog and adding the documentation to the
API docs should be enough.

# Drawbacks

Obviously the API surface area of `Ember.Array` would increase by two methods.
However, I don't believe that this is a real concern since `findIndex` is a
standardized ES6 method. As mentioned before, proficient JavaScript users would
probably even expect this method to be available.

Also, file size would increase slightly.

# Alternatives

## Addon

Instead of baking `findIndex` and `findIndexBy` into the framework, these
methods could also be made available as an addon (as utility functions and a
mixin). The drawback is that usage is a bit less ergonomic. The addon / users
would either have to monkey patch `Ember.Array` with the mixin or use this
rather "unnatural" syntax:

```js
import { A } from '@ember/array';
import { findIndex, findIndexBy } from 'ember-find-index';

const users = A([
  { name: 'Alice', age: 16 },
  { name: 'Bob', age: 18 },
  { name: 'Carol', age: 21 },
  { name: 'Dave', age: 20 }
]);

function isAllowedToDrink(user) {
  return user.age >= 21;
}

// const idxCanDrink = users.findIndex(isAllowedToDrink);
const idxCanDrink = findIndex(users, isAllowedToDrink);

// const idxDave = users.findIndexBy('name', 'Dave');
const idxDave = findIndexBy(users, 'name', 'Dave');
```

## Closure / "leaky" find callback function

Since the callback function receives the index of the current element as the
second argument, users could also "abuse" `find` in this way:

```js
import { A } from '@ember/array';

const users = A([
  { name: 'Alice', age: 16 },
  { name: 'Bob', age: 18 },
  { name: 'Carol', age: 21 },
  { name: 'Dave', age: 20 }
]);

let idxCanDrink = -1;

users.find((user, idx) => {
  if (user.age >= 21) {
    idxCanDrink = idx;
    return true;
  }
  return false;
});

let idxDave = -1;

users.find((user, idx) => {
  if (user.name === 'Dave') {
    idxDave = idx;
    return true;
  }
  return false;
});
```

The disadvantage is that this code is quite verbose and it's no immediately
apparent what the intention is.

# Unresolved questions

- Should the common logic be factored out into a common utility method?
- What should that methods name be?
- Should it return a double tuple or object? (tuple is probably faster)

[mdn-findIndex]: https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Array/findIndex
[ember-array]: https://www.emberjs.com/api/ember/2.14/classes/Ember.Array
[ember-array-find]: https://www.emberjs.com/api/ember/2.14/classes/Ember.Array/methods/find?anchor=find
[ember-array-findBy]: https://www.emberjs.com/api/ember/2.14/classes/Ember.Array/methods/find?anchor=findBy
[ember-enumerable]: https://www.emberjs.com/api/ember/2.14/classes/Ember.Enumerable
[code-find]: https://github.com/emberjs/ember.js/blob/38e9c8965347e94ae9a081794cf2b5dcc0e8f978/packages/ember-runtime/lib/mixins/enumerable.js#L478-L536
[code-findBy]: https://github.com/emberjs/ember.js/blob/38e9c8965347e94ae9a081794cf2b5dcc0e8f978/packages/ember-runtime/lib/mixins/enumerable.js#L538-L553
[code-iter]: https://github.com/emberjs/ember.js/blob/v2.14.1/packages/ember-runtime/lib/mixins/enumerable.js#L45-L54
