---
Start Date: 2018-06-19
RFC PR: https://github.com/emberjs/rfcs/pull/340
Ember Issue: (leave this empty)

---

# Deprecate Ember.merge in favor of Ember.assign

## Summary

The goal of this RFC is to remove `Ember.merge` in favor of using `Ember.assign`.

## Motivation

`Ember.assign` has been around quite awhile, and has the same functionality as `Ember.merge`.
With that in mind, we should remove the old `Ember.merge`, in favor of just having a single function.

## Detailed design

Ember will start logging deprecation messages that tell you to use `Ember.assign` instead of `Ember.merge`.

The exact deprecation message will be decided later, but something along the lines of:

```
Using `Ember.merge` is deprecated. Please use `Ember.assign` instead. If you are using a version of
Ember <= 2.4 you can use [ember-assign-polyfill](https://github.com/shipshapecode/ember-assign-polyfill) to make `Ember.assign`
available to you.
```

## How we teach this

This should be a simple 1 to 1 conversion, and the deprecation message should be clear enough for all to 
understand what they need to do, and convert all usages of `Ember.merge` to `Ember.assign`.

### Deprecation Guide

An entry to the [Deprecation Guides](https://emberjs.com/deprecations/) will be added outlining the conversion from
`Ember.merge` to `Ember.assign`.

`Ember.merge` predates `Ember.assign`, but since `Ember.assign` has been released, `Ember.merge` has been mostly unnecessary.
To cut down on duplication, we are now recommending using `Ember.assign` instead of `Ember.merge`. If you are using a version of
Ember <= 2.4 you can use [ember-assign-polyfill](https://github.com/shipshapecode/ember-assign-polyfill) to make `Ember.assign`
available to you.

Before:
```js
import { merge } from '@ember/polyfills';

var a = { first: 'Yehuda' };
var b = { last: 'Katz' };
merge(a, b); // a == { first: 'Yehuda', last: 'Katz' }, b == { last: 'Katz' }

```

After:
```js
import { assign } from '@ember/polyfills';

var a = { first: 'Yehuda' };
var b = { last: 'Katz' };
assign(a, b); // a == { first: 'Yehuda', last: 'Katz' }, b == { last: 'Katz' }
```

### Codemod

A codemod will be provided to allow automatic conversion of `Ember.merge` to `Ember.assign`.

## Drawbacks

The only drawback, that I can think of, is people would need to convert `Ember.merge` to 
`Ember.assign`, but this would be a very easy change and could easily be done via codemod.

## Alternatives

The impact of not doing this, is we continue to have two functions that do basically the same thing,
which we need to maintain. 

Another alternative, could be to remove both `Ember.merge` and `Ember.assign`, in favor of `Object.assign`
or something similar.

## Unresolved questions

None, that I can think of.
