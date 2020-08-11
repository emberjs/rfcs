- Start Date: 2019-12-22
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/566
- Tracking: (leave this empty)

# @cached

## Summary

Add a `@cached` decorator for memoizing the result of a getter based on
autotracking. In the following example, `fullName` would only recalculate if
`firstName` or `lastName` is updated.

```js
class Person {
  @tracked firstName = 'Jen';
  @tracked lastName = 'Weber';

  @cached
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

## Motivation

One of the major differences between computed properties and tracked properties
with autotracking in Octane is that native, autotracked getters do not
automatically cache their values, where computed properties were cached by
default. This was an intentional design choice, as the memoization logic for
computed properties was actually more costly, on average, than rerunning the
getter in the first place. This was especially true given that computed
properties would usually only ever be calculated and used once or twice per
render before being updated.

However, there are absolutely cases where getters _are_ expensive, and their
values are used repeatedly, so memoization would be very helpful. Strategic,
opt-in memoization is a useful tool that would help Ember developers optimize
their apps when relevant, without adding extra overhead unless necessary.

## Detailed design

The `@cached` decorator will be exported from `@glimmer/tracking`, alongside
`@tracked`. It can be used on native getters to memoize their return values
based on the tracked state they consume while being calculated.

```js
import { tracked, cached } from '@glimmer/tracking';

class Person {
  @tracked firstName = 'Jen';
  @tracked lastName = 'Weber';

  @cached
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

In this example, the `fullName` getter will be memoized whenever it is called,
and will only be recalculated the next time the `firstName` or `lastName`
properties are set. This would apply to any autotracking tags consumed while
calculating the getter, so changes to `EmberArray`s and other tracked
primitives, for instance, would also cause invalidations.

If used on a non-getter, `@cached` will throw an error in `DEBUG` modes.
Properties can also include a setter, but it won't affect the memoization of the
getter (except by potentially setting the state that was tracked in the first
place).

### Invalidation

`@cached` will propagate invalidations whenever any of the properties it is
entangled with are invalidated, causing any downstream state that has consumed
to be invalidated at the same time.

This is necessary in order to have the memoized value be pulled on again in
general. There is no way to, for instance, only propagate changes if the
memoized value has changed, since we must calculate the memoized value first to
know what it's new value is, and we must propagate the change in order to ensure
that it will be recalculated.

### Cycles

Cycles will not be allowed with `@cached`. The cache will only be activated
_after_ the getter has fully calculated, so any cycles will cause infinite
recursion (and eventually, stack overflow), just like un-memoized getters. If a
cycle is detected in `DEBUG` mode, it will throw an error.

## How we teach this

`@cached` is not an essential part of the reactivity model, so it shouldn't be
covered during the main component/reactivity guide flow. Instead, it should be
covered in the intermediate/in-depth guides, and in any performance related
guides.

### API Docs

TODO

## Drawbacks

- Adds extra complexity when programming (whether or not a value should be
  memoized is now a decision that has to be made). In general, we should make
  sure this is not an issue by recommending that memoization idiomatically _not_
  be used _unless_ it is absolutely necessary.

- Adds extra overhead for each memoized getter. This again should be addressed
  by teaching that it should be avoided when possible. The cost tradeoff should
  be noted in documentation in particular to emphasize this, and discourage
  overuse.

- `@cached` may rerun even if the values themselves have not changed, since
  tracked properties will always invalidate even if their underlying value did
  not change. Unfortunately, this is not really something that `@cached` can
  circumvent, since there's no way to tell if a value has actually changed, or
  to know which values are being accessed when the memoized value is accessed
  until the getter is run.

  Instead, we should be sure that the rules of property invalidation are clear,
  and in performance sensitive situations we recommend diff checking when
  assigning the property:

  ```js
  if (newValue !== this.trackedProp) {
    this.trackedProp = newValue;
  }
  ```

## Alternatives

- `@memo` or `@memoized` could be alternative names. Initially `@memo` was used,
  but it was decided to switch to `@cached`.

- `@cached` could receive arguments of the keys to memoize based on. This would
  bring us back to the ergonomics of computed properties, however, and would not
  be ideal. It also would bring no actual benefits, except being able to exclude
  certain values from recalculation.


