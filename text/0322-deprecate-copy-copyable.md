---
2018-03-24: 
RFC PR: 322
Ember Issue: (leave this empty)

---

# Deprecation of Ember.copy and Ember.Copyable

## Summary

This RFC recommends the deprecation and eventual removal of `Ember.copy` and the `Ember.Copyable` mixin.

## Motivation

A deep-copy mechanism is certainly useful, but it is a general JavaScript problem. Ember itself doesn't need to offer one, especially one that Ember itself isn't using internally. This function and its accompanying mixin arrived with SproutCore, a long time ago, and are not used by Ember itself, even though they currently reside in `@ember/object/internals`.

`ember-data` uses `Ember.copy` to do deep-copies. However, the `ember-data` team finds its needs would be better served by a private deep-copy mechanism that doesn't flow inadvertently through external interfaces into the `Ember.copy` methods of user-supplied objects. These interfaces are not designed to support deep copies of user-supplied data, and it can raise havoc in the form of hard-to-diagnose bugs, especially in test scenarios.

Since `ember` and `ember-data` do not intend to use this mechanism going forward, it would be better to remove it from the Ember codebase and extract it into an add-on for those who wish to continue to use it.

## Detailed design

There are four steps to deprecating any function:
* logging the deprecation in the call
* removal of calls to the function from ember and any add-ons that ship with ember-cli
* extraction to an add-on
* eventual removal of the feature in the stated release (in this case 4.0.0).

This RFC deprecates the `copy` function and `Copyable` mixin of `@ember/object/internals`.

Shallow copies of the form  `copy(x)`  or `copy(x, false)` can be replaced mechanically with `Object.assign({}, x)`. The simplest way to deal with deep copies in any situation depends upon the nature of the data involved.

### Current internal uses

#### `ember-source`

This following modules in `packages/ember-runtime/lib` implement the code being deprecated:

* `copy.js` contains the `copy()` function that will log the deprecation before executing,
* `mixins/copyable.js` provides the `Copyable` mixin, but it contains no executable code to deprecate.
* `mixins/array.js` - The `NativeArray` mixin extends the `Copyable` mixin and implements `copy()`.

The following tests in `packages/ember-runtime/tests` use the implementation above:

* `core/copy_test.js` tests the `copy()` method itself.
* `copyable-array/copy-test.js` tests the `copy()` method of a `NativeArray` for identical results.
* `helpers/array.js` provides the arrays used by the `NativeArray` test above.
* `system/native_array/copyable_suite_test.js` tests the independence of the results of deep copying a `NativeArray`

The route  `packages/ember-routing/lib/system/route.js` has one shallow copy, but the test  `packages/ember/tests/routing/decoupled_basic_test` is using deep copy.

The `copy()` methods in `packages/ember-metal/lib/map.js` and  `chains.js` and their use in `meta.js`, and  `map_test.js` are unrelated.

At present, the handling of arrays in `Ember.copy` is inconsistent. `NativeArray` uses the `Copyable` mixin and implements a `copy` method. When calling `Ember.copy`, passing a `NativeArray`, it will note that the passed parameter uses `Copyable` and call the copy method inside  `NativeArray`. However, the recursive `_copy` method that `Ember.copy` calls for other objects has its own generic mechanism for copying arrays. If `copy` is passed a non-`Copyable` object that contains a `NativeArray` as a member, when the recursion gets to that member, it will use the generic mechanism rather than delegating to the `copy` method within the  `NativeArray`.

The recursive  `_copy` method also has an assertion that will fail if it is called with any `EmberObject` that is not also `Copyable`. This assertion occurs before (and hence affects) the code which handles arrays, even though, for arrays, the object's `copy` method isn't then used.

During the deprecation period, the `Ember.copy` method and the `NativeArray.copy` methods will carry a deprecation warning. We will remove `Copyable` from `NativeArray` and change  `Ember.copy` to consistently use the common array copy mechanism to copy arrays rather than sometimes delegating. We will move the assertion that an `EmberObject` must be  `Copyable` to the clause that handles non-array objects.

We need a way to deprecate use of the `Copyable` mixin. If the penalty for adding code in such a common place isn't too high, we could have `core_object.extend()`  check for `Copyable` and deprecate accordingly. We will also supply a new eslint warning that flags the deprecated use of `Copyable`. (This may be our first eslint check for deprecations. We may want to consider adding others at the same time.)

Those using the add-on will need to mechanically adjust any uses of  `myArray.copy(deep)` to  `copy(myArray, deep)` in order to avoid the deprecation message.

At the end of this period, we will remove the deprecated copy() method, the Copyable mixin, and the deprecated NativeArray.copy() method.

#### `ember-data`

The following code in `ember-data` uses `copy()`, but only for shallow copies:
* `addon/-private/system/model/internal-model.js` - one use
* `addon/-private/system/snapshot.js` - two uses
* `addon/-private/system/store.js` - one use

All of the following uses in tests perform deep copies:
* `tests/integration/adapter/build-url-mixin-test.js` - two uses
* `tests/integration/adapter/rest-adapter-test.js` - two uses
* `tests/integration/store-test.js` - two uses
* `tests/unit/system/relationships/polymorphic-relationship-payloads-test.js` - four uses

The `copy()` methods referenced in `addon/-private/system.map.js` and  `addon/-private/system/relationships/state/relationship.js` are unrelated.

It would appear that deep copy is used within these packages only during testing, and generally to ensure fresh test data without side-effects.

### Current external uses

The key considerations for add-ons or apps looking for an alternative to copy() and Copyable are:
* Do they call `copy()` to do shallow copies or deep copies?
* If deep copies are being performed, are the objects involved POJOs or are they derived from `EmberObject`?
* Do they provide objects that use the `Copyable` mixin with `copy()` methods intended for use in deep copies by other classes?
* Is the data you are copying the sort of thing where you can do the copy in its behalf, or does it require collaboration from the object itself? Or are the contents so open-ended that you can't possibly know?

Shallow copies are directly supported by ES6. It's easy to perform recursive deep copies for most simple POJOs without delegating work to the object you are copying. For more complex data, you may need some kind of recursive delegation. `Copyable` is a delegation mechanism, and apps and add-ons that require delegation will probably want to use the proposed add-on.

The Code Search capabilities of emberobserver are a wonderful way to get a glimpse of how code in the wild is using particular features.

A quick search of the top-scoring add-on packages revealed that most, but by no means all, of the uses of `copy()` in the modules were for shallow copies that can be accomplished using Object.assign, so a lot of the code affected by this deprecation can rely on a simple substitution.

Very few packages used `Copyable` - only 9 across the whole set - and most used the feature for only one class.   `ember-data-copyable` is probably most wedded to the mechanism: it delivers a  `Copyable`-based mixin for asynchronous copying.  `ember-data-model-fragments` has pretty open-ended properties. These add-ons would be likely to use the proposed add-on moving forward.   `ember-restless`, and `ember-calendar` appear more bounded. Any deep copy mechanism for POJOs may meet their needs.

### Add-on

The add-on will supply the `copy()` function and the `Copyable` mixin based on the existing code, modified as indicated above for handling of arrays.

We could treat the add-on as the extraction of a feature from the monolithic `ember-source`, as was recently done for strings. If we choose to frame it in that way, the naming should follow the conventions set out for extracting elements of Ember into their own packages. If we choose not to frame it that way, then naming is one of the things this section should specify clearly.

## How we teach this

### Communication of change

We need to inform users that `Ember.copy` and `Ember.Copyable` will be dprecated and in what release it will occur. This notification should also point them to the add-on for those who need it.

### Official code bases and documentation

We do not actively teach the use of `Ember.copy`. It doesn't appear anywhere in our guides, website, or tutorial. Once it is gone from the code, we also need to verify it no longer appears in the API listings.

We must provide an entry in the deprecation guide for this change:
* describing the use of `to = Object.assign({},from)` for shallow copies.
* pointing out viable alternatives for deep copies.
* directing heavy users of deep copies to the addon.

## Drawbacks

The primary drawback is the API churn of people pulling it out of their code. However, for most uses, the change will be straightforward, and the add-on will be available for the foreseeable future for those who want to continue with the implementation.

## Alternatives

We could simply leave it in place as a utility for others to use. Even then, it would make sense to split it out into its own module, as has already been done for strings, so the work would be much the same.

## Unresolved questions

None at the moment...


