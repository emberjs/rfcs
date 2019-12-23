- Start Date: 2019-12-23
- Relevant Team(s): Ember.js
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# TrackedList

## Summary

Introduces a new array-like class whose usage is autotracked by default, and
is designed with performance in mind.

```js
import { TrackedList } from '@glimmer/tracking';

class Person {
  friends = TrackedList.from(['Yehuda', 'Melanie', 'Ricardo']);

  addFriend(newFriend) {
    this.friends.push(newFriend);
  }
}
```

## Motivation

`@tracked` in Ember Octane was primarily designed around the use case of
tracking changes to _properties_ on classes. However, there are other forms of
state that need to be changed in an Ember app, and the most common one is
tracking changes to _arrays_.

When `@tracked` was introduced, one of the suggested possibilities for tracking
changes to arrays was the _immutable pattern_:

```js
class Person {
  @friends = ['Yehuda', 'Melanie', 'Ricardo'];

  addFriend(newFriend) {
    this.friends = [...this.friends, newFriend];
  }
}
```

This pattern is solid and works very well for many Octane users, but it is
fairly opinionated toward a functional-immutable style of programming, one that
is different from many Ember apps historically, and that conflicts with
established programming styles in the ecosystem. This is not necessarily a bad
thing - one of autotracking's strengths is that it is unopinionated about
programming style, and can be used with many different paradigms - but it means
that users who prefer an Object Oriented Programming (OOP) style don't have a
modern, ergonomic equivalent.

We received feedback to this effect from a good number of Octane early adopters,
which is why we decided to switch back to using `EmberArray` within the guides
and documentation for our basic examples.

```js
import { A } from '@ember/array';

class Person {
  @friends = A(['Yehuda', 'Melanie', 'Ricardo']);

  addFriend(newFriend) {
    this.friends.pushObject(newFriend);
  }
}
```

`EmberArray` is reactive, so changes to it will update autotracked values and it
has the behaviors most OOP users want. However, it has a lot of its own baggage:

- KVO methods like `pushObject` and `popObject` have to be learned in order to
  use the array, which means learning a whole new array API to be effective in
  Ember.
- Custom extra methods like `mapBy` and `sortBy` add to the confusion, since
  they extend the core array API, and sometimes have conflicting names (e.g.
  `any` vs `some`)
- Prototype extensions can still be used, and it can be confusing to new users
  since their DX is fairly nice, even though they have many downsides and are
  generally recommended against.
- Performance is not great, since they either rely on patching the array
  prototype, which is very bad, or on re-implementing most array methods under
  the hood.

### Introducing `TrackedList`

The goal of `TrackedList` is to provide a better `EmberArray`, in the short
term, for Ember users who prefer this style of programming; One that lines up
more directly with standard array behaviors, and is also reactive.

```js
import { TrackedList } from '@glimmer/tracking';

class Person {
  friends = TrackedList.from(['Yehuda', 'Melanie', 'Ricardo']);

  addFriend(newFriend) {
    this.friends.push(newFriend);
  }

  get firstFriend() {
    let [first, ...rest] = this.friends;

    return first;
  }
}
```

`TrackedList` will be as close to 1-1 with the native array API as possible.
However, it will not attempt to be a drop-in replacement for native arrays,
since this is not possible without native `Proxy` (which is not supported in
older browsers) and would have major performance caveats even if it were. This
is also the motivation behind the name: `TrackedArray` would imply a tracked
version of native arrays, where `TrackedList` implies that it is its own data
structure, with potentially divergent behaviors.

## Detailed design

`TrackedList` will re-implement the array API 1-1 for the most part, including
native iterables in environments where they are available (enabling native
destructuring syntax). Any existing prototype and static methods on `Array` will
also exist on `TrackedList`, and future methods that are added will be added as
soon as possible, without the need for an RFC (unless it would require
significant changes to the way `TrackedList` works).

As such, the rest of the design portion of this RFC will cover the tracking
dynamics of `TrackedList`, and the ways in which it diverges from native arrays.

### Tracking Dynamics

In general, any changes to a `TrackedList` will be propagated as changes to the
_whole_ list, rather than to individual indexes within the list. This is for two
main reasons:

1. Most changes will result in downstream consumers having to reprocess the
   entire list, in general.
2. It is much more performant to track changes to the list as a whole rather
   than to each individual item within the list.

In the following example, for instance, the `firstFriend` getter would be
invalidated every time a new friend is added, even though it only accesses the
first item in the list:

```js
import { TrackedList } from '@glimmer/tracking';

class Person {
  friends = TrackedList.from(['Yehuda', 'Melanie', 'Ricardo']);

  addFriend(newFriend) {
    this.friends.push(newFriend);
  }

  get firstFriend() {
    let [first] = this.friends;

    return first;
  }
}
```

From a change tracking perspective, we can see that this is actually very
similar to the immutable pattern described at the beginning of this RFC - we
are essentially telling the tracking system that some changes have occured, and
now `friends` is _effectively_ a new array that needs to be reprocessed.

In general, methods that _access_ the list and don't mutate it will entangle
the list in the autotracking context. Methods that _mutate_ the list will
invalidate, but not entangle the list, preventing backflow issues when
rendering.

### `getAt` and `setAt`

It will not be possible to access the list items directly using `[]` syntax,
like on normal arrays, since this would require native proxies (and would incur
performance penalties). `TrackedList` will have two extra methods instead for
direct index access: `getAt` and `setAt`. These methods will operate similar to
`objectAt` and `replace` on `EmberArray` today, allowing the user to pass a
numeric index as the first value, and in the case of `setAt` to pass the value
to set the index to as the second:

```ts
declare class TrackedList<T> {
  getAt(index: number): T;
  setAt(index: number, value: T): T;
}
```

### `sort` and `reverse`

`sort` and `reverse` are unusual array methods, because they both _read_ the
value, _and_ mutate the array, in place. Most of the time, users want to do one
or the other, and from a change tracking perspective this is not ideal behavior.

As such, `TrackedList` will instead always return a new list when either of
these methods are called, allowing users to derive state without accidentally
mutating upstream values.

### `arr` in Array Callbacks

Many array callbacks, such as `map` and `reduce`, receive the array as the third
argument:

```js
[1, 2, 3].map((value, index, arr) => value + 1);
```

For performance reasons, `TrackedList` will not provide the list itself as the
last argument. Instead, it will provide the underlying array, which is used as a
storage mechanism to back the list. In `DEBUG` mode, this array will be wrapped
with a Proxy which will:

1. Alert users if they attempt to use non-standard methods such as `getAt` and
   `setAt`.
2. Throw an error if users try to mutate the array in any way. It should be
   considered read-only for the purposes of usage within callbacks.

This will allow common array-mapping libraries to be used interoperably with
lists, without causing issues by accidentally mutating the internal storage of
the list.

### `Array.isArray`, `isEmberArray`, and Instance Checks

`Array.isArray` and `isEmberArray` will return `false` for instances of
`TrackedList`. Users can check to see if it is an instance using `instanceOf`:

```js
let friends = new TrackedList();

friends instanceOf TrackedList; // true
```

### `length`

`length` will be an immutable/readonly property on `TrackedList`. This is to
prevent complications around sparse lists and sparse list iteration, and keep
lists simpler overall. This restriction could be lifted in future RFCs.

### `Ember.A`

`Ember.A` will do nothing if it receives a `TrackedList`, and return the
original value instead.

## How we teach this

`TrackedList` will be the default reactive array class recommended moving
forward, and is meant to replace `EmberArray` entirely. The main guides should
be updated to reflect this. In the advanced guides, there should be a section
covering the functional-immutable style of updating arrays and objects.

### API Docs

TODO

## Drawbacks

- Introduces a new array-like class, which may cause confusion as the community
  transitions.

- Introduces an array class that _may_ have a limited shelf life. IE11 support
  will eventually be removed, and allow us to use native `Proxy` to
  transparently wrap real native arrays instead. However, this potential class
  would have a few of its own caveats:
  - Performance. Proxies have gotten much better, but they are still much less
    performant compared to native arrays.
  - General interop concerns. Arrays have a lot of odd behaviors that have been
    built up over the years, such as the behaviors of sparse arrays, the
    differences between how numeric and string based keys are handled, etc.
  Given these caveats, and the fact that IE11 is still supported and will be for
  the indeterminate future, a minimal array-like class that improves on the
  existing DX seems like a good middle-ground solution for the time being.

## Alternatives

- `getAt` and `setAt` could be changed to match the `objectAt` and `replace`
  methods that exist on `EmberArray`. This would be better for interoperability,
  but comes with caveats:
  - `objectAt` is not accurate for values that contain primitive values
  - `replace` is a not an ideal method for updating a value at a predetermined
    index. It's API is more similar to `splice`, which is confusing and
    difficult to work with.
  Some potential alternatives include:
  - `objectAt` and `replaceAt`
  - `valueAt` and `replaceAt`
  - `valueAt` and `setValueAt`
  `TrackedList` could also implement `objectAt` and `replace` in order to remain
  mostly compatible with legacy code, but recommend that `getAt`/`setAt` be used
  by convention.

- Passing the underlying array as the third argument to methods like `map` and
  `forEach` may be confusing to users. However, passing the list itself instead
  would remove many of the browsers optimizations for these methods. They are
  implemented in most modern runtimes as precompiled, pre-JITed code, and
  matching anything near the level of native performance is likely impossible.
  The other alternative would be to not pass a third argument at all (in
  development mode), which would prevent users from relying on it in the first
  place. This would limit the interoperability of `TrackedList` somewhat, but
  would likely not have a massive impact.

## Unresolved questions

- How does this impact the potential functional array-methods RFC (functional
  `objectAt` and `replace`, etc)?
- Should `EmberArray` implement `getAt` and `setAt` to match the new API for
  getting/setting indexes?
