- Start Date: 2017-06-29
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Deprecate `Enumerable#any` and introduce `Enumerable#some`.

# Motivation

[`any`](https://emberjs.com/api/classes/Ember.Array.html#method_any) has been
implemented on `Ember.Enumerable` for years, however native javascript arrays have
`Array.prototype.some` to perform the same task.

This RFC proposes the addition of `Ember.Enumerable#some`, the aliasing of the old `Ember.Enumerable#any` to it,
and encourage the usage of the first either in the docs or with deprecation to be as aligned as possible
with native JavaScript arrays.

# Detailed design

The first step is to rename `any` to `some` and recreate `any` as simply an alias
of the new method.

The second and optional step is to encourage, possibly with a deprecation, the usage of `some`
over `any`.

`Ember.Enumerable` is a mixin part of `Ember.NativeArray`, which is applied to `Array.prototype`
if the user doesn't disable prototype extensions. We do not want to replace the native `Array.prototype.some`
with ours, possibly slower, JavaScript version so it has to be implemented on a way that respects the native
implementation.

# How We Teach This

The documentation of [`any`](https://emberjs.com/api/classes/Ember.Array.html#method_any) already
states that it is equivalent to `Array.prototype.some`. The deprecation and the entries on the guide
should make it easier, since the semantics of the new function are identical.

There has been [a similar RFC to deprecate `contains` in favour of `includes`](https://github.com/emberjs/rfcs/pull/136)
in the past and it wasn't a deprecation that caused much rejection from the community.

# Drawbacks

Adding the new function is pretty harmless. It only extends the public API surface a little bit, but arguably
the second part of the RFC, deprecate `any` in favour of `some` will surely add a certain amount
of churn for developers.

# Alternatives

Do not add the feature or, in case the feature is seen as positive but the churn is considered too harmful,
do not deprecate `any`.

# Unresolved questions

There is also a close cousin on `any`, and that is the `isAny(key, value)` convenience function. While
this function does not clash with any native function, it could be a good idea to rename it to `isSome`
for consistency.
