- Start Date: 2015-07-31
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

At the moment, the setter function in computed properties caches the return value of the function,
even when the return value is undefined. While undefined is a perfectly valid value, cases where
you really want to cache undefined are very rare, and most of the time it's just the user forgetting
to return from the function.

I think there is a better (or even completely) retrocompatible way to improve user experience
and be closer to regular javascript.

# Motivation

I've found that computed properties with getters and setters keep biting newcomers because they
don't behave like regular JS getters and setters, and I think they can be more user friendly and
also more aligned with regular javascript.

This is the kind of code that keeps confusing users:

```js
height: 100,
goldenRatioWidth: computed('height', {
  get() {
    return this.get('height') * 1.618;
  },
  set(_, value) {
    this.set('height', value / 1.618);
  }
})
```

I've explained myself half a dozen times in the last month to confused programmers why this
snippet doesn't work as expected, and it's easy to see why they are confused. The equivalent
code with regular getters and setters would look like this:

```js
height: 100,
get goldenRatioWidth() {
  return this.height * 1.618;
},
set goldenRatioWidth(value) {
  this.height = value / 1.618;
}
```

No return, but the code works as expected because there is no caching, and the next time the
property is accessed the getter will be invoked and the expected result will be computed.

I think that computed properties should be as close to this behavior as they possibly can,
especially when code with decorators will look like this:

```js
@computed('height')
get goldenRatioWidth() {
  return this.height * 1.618;
},
set goldenRatioWidth(value) {
  this.height = value / 1.618;
}
```

In the snippet above I'd expect the code to behave the same with and without the decorator except
for the "caching with declarative expiration policy" that the decorator adds.

# Detailed design

The reason why people get confused is that most people doesn't expect the setter to cache.
Although your getter and setter can do any crazy stuff, in most sanely defined computed properties
developers look for "idempotence" (not sure if it's the better term). When you invoke
`set('myProp', 200)` they expect the next time they get the property with `get('myProp')` to
get the same value they just set.

When someone implements a setter in a computed property, it's because setting state in this
computed property:

A) Has to set state in another property, like the example above. Typically that other property
is one of the dependent keys.

B) Has to trigger some callback-like logic. Typically this is way of replacing an observer.

In computed properties for case A, after setting the state in a dependent key the users, used to
the way ES5 getters/setters work, think that they're done, because the next time the property is
accessed the getter will be invoked and the expected result will come out.

And it doesn't because setters cache `undefined` as return value. I dare to affirm that is almost never
the indended behaviour.

One radical solution would be not to cache at all in setters (getters would cache as usual).
This is a rather aggressive change, although I think that "idemponent" computed properties
(the big mayority) the getter would return the exact same value and nobody would notice the
difference, but in a few use cases where users use the cache to just store a calculation that
would be a pain, forcing users to store that same value in a `_private` key by example.
On the bright side, this is as close to ES5 getters/setters as you can posibly be.

A mostly retrocompatible option would be to **not cache undefined** from the setters. If the user
doesn't return from a setter, nothing is memoized, the property is just marked as stale
and the next access will invoke the getter. Getters would continue to cache undefined as they do now.
If a user returns a value, it works as usual. However `return undefined;` won't cache the value.
That is the only non retrocompatible change. Computed properties that exist in the wild right now
will all have a explicit return and won't be affected.

@stefanpenner in slack suggested an improvement to the second approach that would make this
completely retrocompatible in exchange for a bit of computational cost. When the setter returns
`undefined` the can do `setter.toString().indexOf('return')` to determine if it was
intended or not. Not sure if the case of caching `undefined` from a getter is common enough to
justify this, and how unsafe it is to do that (it can be edge cases with the detection of explicit return).

Another 4th option that might enable this is wait for decorators to land. If there is enough reasons to take
this approach, the `@computed('foo', 'bar') + <getter> & <setter>` decorator can have slightly different (better) 
semantics than `computed('foo', 'bar', { get() { }, set(_, v){  }  })`. It's new syntax after all, so it's not
technically a breaking change.

# Drawbacks

The first approach is not retrocompatible and makes some use cases less straightforward.

The second approach is mostly retrocompatible, but not completely.

The third approach is is probably completely retrocompatible, at the expense of being computationally more expensive.

The forth approach introduced new semantics when using the `@computed` decorator, so it's not technically a breaking 
change, but both old and new computed properties have to coexist until Ember 3.0 and it has to be very well communicated
, documented, twitted and blogged so people understand that `@computed` and `computed(fn)` are not entirely equivalent.

# Alternatives

Leave the bahaviour as it is right now and just educate users. This is what I've been doing but people
still get confused, so clearly we have to try harder.


# Unresolved questions

Is it possible to detect if a function does explicitly return in "compile time"? That could make
possible a completely retrocompatible solution with no performance cost.
