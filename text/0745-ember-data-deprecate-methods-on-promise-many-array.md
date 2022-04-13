---
Stage: Accepted
Start Date: 2021-04-23
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): ember-data
RFC PR: https://github.com/emberjs/rfcs/pull/745
---

# EmberData | Modernize PromiseManyArray

## Summary

Deprecates reading, mutating or operating on an async `hasMany` relationship before
resolving it's value in the application's Javascript code. Rendering an async `hasMany`
in a template will continue to work as expected.

## Motivation

`PromiseManyArray` is a promisified `ArrayProxy` that serves as a fetch wrapper when
a user either implicitly or explicitly defines a `hasMany` relationship as async. It
is the synchronous return value when a user access the async relationship on a record instance.

For instance, given the following `Post` model.

```js
class Post extends Model {
  @hasMany('comment` { inverse: null, async: true })
  comments
}
```

A user today might access and map over the comments for a post in the following manner.

```js
const commentProxy = post.comments;
const commentIds = commentProxy.map(comment => comment.id);
```

This often works because any `resolved` data for the relationship is immediately accessible
via the `ArrayProxy`. However, if the network request initiated by accessing the comments
relationship results in changes to what records are available, those changes would not be
reflected in the `commentIds`. More often, users will `await` the relationship to ensure the
data is fully ready for use.

```js
const comments = await post.comments;
const commentIds = comments.map(comment => comment.id);
```

As the language and framework have evolved, better features have become available that
negate the value of many of the methods that exist on `PromiseManyArray`. This RFC seeks
to better align the way folks interact with an async relationship with the `Octane` paradigm,
`tracked` properties, and `async/await`. Doing so allows us to take a major first step towards
simplifying the use of array-like proxies throughout EmberData.

## Detailed design

This RFC will reduce the API surface area of PromiseManyArray to just those methods needed to
serve it's primary functions.

- as a thenable/awaitable value (with promise state flags)
- as a custom enumerable that for-convenience can be directly iterated by templates.

All other methods and properties on PromiseManyArray would be deprecated. Additionally,
the methods `addArrayObserver` and `removeArrayObserver` on `ManyArray` will be deprecated.

Code examples, guides, and documentation should encourage users to resolve the relationship
before operating on it, including avoiding the use of the remaining iterable APIs (which will
be marked private).

This deprecation would target `5.0` and would become `enabled` no-sooner than `4.1` (although
it may be made `available` before then).

We will keep the following properties and methods previously supplied by the `PromiseProxyMixin`

- the public promise state flag `isSettled`
- the public promise state flag `isPending`
- the public promise state flag `isRejected`
- the public promise state flag `isFulfilled`
- the private property `promise`
- the method `then`
- the method `catch`
- the method `finally`

But we will eliminate the private property `reason`.

We will keep the following property we had supplied on our own

- the public property `links`
- the method `reload`

While deprecating the following methods which delegated to the ManyArray

- `createRecord`

As `PromiseManyArray` historically extended `ArrayProxy`, it inherited a large API surface from
`MutableArray` `EmberArray` and `EmberObject` the totality of which will be deprecated *except* for:

- the private property `content`
- the property `length`
- the public property `isDestroyed`
- the iterator method `objectAt` (but which will now be marked private)


### Finalized API of PromiseManyArray

Remaining API surface

- `content` (private)
- `length`
- `isDestroyed`
- `objectAt` (private)
- `links`
- `reload`
- `promise`  (private)
- `isSettled`
- `isPending`
- `isFulfilled`
- `isRejected`
- `then`
- `catch`
- `finally`

Deprecated API surface

- `reason`
- `createRecord`
- `firstObject`
- `lastObject`
- `addObserver`
- `cacheFor`
- `decrementProperty`
- `get`
- `getProperties`
- `incrementProperty`
- `notifyPropertyChange`
- `removeObserver`
- `set`
- `setProperties`
- `toggleProperty`
- `addArrayObserver`
- `addObject`
- `addObjects`
- `any`
- `arrayContentDidChange`
- `arrayContentWillChange`
- `clear`
- `compact`
- `every`
- `filter`
- `filterBy`
- `find`
- `findBy`
- `forEach`
- `getEach`
- `includes`
- `indexOf`
- `insertAt`
- `invoke`
- `isAny`
- `isEvery`
- `lastIndexOf`
- `map`
- `mapBy`
- `objectsAt`
- `popObject`
- `pushObject`
- `pushObjects`
- `reduce`
- `reject`
- `rejectBy`
- `removeArrayObserver`
- `removeAt`
- `removeObject`
- `removeObjects`
- `replace`
- `reverseObjects`
- `setEach`
- `setObjects`
- `shiftObject`
- `slice`
- `sortBy`
- `toArray`
- `uniq`
- `uniqBy`
- `unshiftObject`
- `unshiftObjects`
- `without`

## How we teach this

As this deprecation targets 4.x, users would need to have upgraded to Octane paradigms before
they could encounter the deprecations listed here. This means for *most* apps this should be as 
trivial as adding an `await` where necessary.

## Drawbacks

It has become fairly common to interact with async hasMany relationships as if they are 
synchronous, and it is not easy to programmatically codemod a conversion. This may result in
a sizeable amount of "find the missing await" for some applications.

## Alternatives

Deprecate array-like APIs on ManyArray simultaneously. Why not? While this is something we seek to do as a follow up, that migration is a more difficult one than this which *only* pushes users to resolve async before interacting with the value. As the ManyArray has all these same methods in a non-deprecated state, existing code will continue to run as expected and without deprecation so long as that resolution is done. This is a first step which will make that deprecation easier when
the time comes.
