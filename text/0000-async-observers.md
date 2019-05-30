- Start Date: 2019-05-30
- Relevant Team(s): Ember.js, Learning
- RFC PR:
- Tracking: (leave this empty)

# Async Observers

## Summary

Add a way to specify whether or not observers should fire synchronously -
that is, immediately after the property they are observing has changed - or
asynchronously during the next runloop, along with an optional feature to
specify whether observers should default to sync or async.

## Motivation

Observers have been run synchronously in Ember since before v1.0 was released,
and for about as long it has been an intention of the core team to eventually
make them asynchronous. There are a two main reasons for why triggering
observers asynchronously would be better overall:

- They promote better programming practices. Synchronous observers can be used
  in a lot of ways to interact with the code they are observing, which puts more
  code on the "hot-path" and is prone to create a mess of intertangled, loosely
  related code filled with [spooky action at a distance](<https://en.wikipedia.org/wiki/Action_at_a_distance_(computer_programming)>).
- It would allow us to clean up a significant chunk of code within Ember
  itself. There is non-trivial amount of code dedicated to sending change
  signals synchronously, and that code has been slowly replaced by an
  alternative system that is lazy. Asynchronous observers would allow us to
  remove legacy code and tech debt.

We implemented this change behind a feature flag, and several community members
tested it out in their applications. In testing, we found that this was
unfortunately too much of a breaking change to do all at once - like it or not,
the timing semantics of observers are public API.

The proposed solution now is to provide a method for users to specify whether an
observer should be sync or async. In existing apps, observers can be converted
incrementally to be async, giving them a path forward. In addition, an optional
feature will be made which sets observers to be async by default, allowing users
to set the default once their whole app has been converted, and allowing new
apps to prevent/discourage sync observers in the first place. In the long run,
synchronous observers will be deprecated and removed.

## Detailed design

### New APIs

A new `sync` boolean argument will be added to both `addObserver` and
`removeObserver`:

```ts
export function addObserver(
  obj: any,
  path: string,
  target: object | Function | null,
  method?: string | Function,
  sync = SYNC_DEFAULT
): void;

export function removeObserver(
  obj: any,
  path: string,
  target: object | Function | null,
  method?: string | Function,
  sync = SYNC_DEFAULT
): void;
```

The argument needs to be added to both because sync and async observers are
tracked separately, so we need to know where to look for the observer when
removing it. Attempting to add both a sync and async observer will throw an
error.

In addition, a new overloaded form of `observer` will allow users to specify
whether or not the observer should be sync or async:

```ts
type ObserverDefinition = {
  dependentKeys: string[];
  fn: Function;
  sync: boolean;
};

export function observer(...args: (string | Function)[]): Function;
export function observer(definition: ObserverDefinition): Function;
```

Users will have to provide a full `ObserverDefinition` to set `sync`, which will
prevent us from having to do any more argument munging to figure out what the
user wants.

### Synchronous Observer Implementation

Since chains are removed, the only way to check if observers should fire is to
cycle through all of them. This means that on every `notifyPropertyChange`, we
will cycle through _all_ active synchronous observers and fire any that have
dirtied.

In apps that are observer heavy, this could lead to performance impacts.
Unfortunately, there isn't much we can do about this. We will try to minimize
the impact as much as possible, but in the end it will be up to individual
applications to migrate away from synchronous observers over time.

### Tracked Properties and `@dependentKeyCompat`

Tracked properties and `@dependentKeyCompat` marked getters/setters will _not_
fire observers synchronously, since they do not use `notifyPropertyChange` or
the old change tracking system at all. In this way, they will encourage users to
convert to async observers, or away from observers entirely.

### Optional Feature

The name of the feature will be `default-async-observers`. Enabling it will
default all observers to be async, but still allow users to set observers to be
synchronous manually.

## How we teach this

### API Docs

(To be added at the [end of the current API docs](https://github.com/emberjs/ember.js/blob/4a98e1610b795edb544513f10a8870af1375141d/packages/%40ember/-internals/runtime/lib/mixins/observable.js#L359))

#### `sync`

By default in new Ember applications, observers are asynchronous. They can be
marked as _synchronous_ instead by using the `sync` option. Synchronous
observers will run immediately when the property they are observing changes,
instead of being scheduled to run later.

Each synchronous observer has a performance impact for every property change, so
you should generally avoid using synchronous observers.

In older applications, observers are synchronous by default. You can use the
`sync` option to make them asynchronous instead and convert them over time. You
can also enable the `default-async-observers` optional feature to make them
asynchronous by default, once you are sure that they will continue to function
if they are asynchronous.

### Guides

Observers are not discussed in the post-Octane guides, since we don't want to
encourage their use. It may make sense to include a section on them in the
upgrade guide instead.

## Drawbacks

The biggest potential drawback is in performance. While we haven't been able to
do any testing on apps that have observers, its possible that these changes will
have an impact on them, especially apps that have many observers.

In theory, this shouldn't impact the majority of Ember apps since observers have
been discouraged so heavily for such a long time. The impact should also
decrease in time, as users transition away from observers entirely and toward
tracked properties.

## Alternatives

- We could release Ember v4, and ship asynchronous observers as a breaking
  change. We currently believe this would be a breaking change that would
  prevent many users from adopting Octane or transitioning forward to tracked
  properties, which would be problematic and could divide the community.

## Unresolved questions

What is the exact performance impact? Can we test it out in an application that
represents a typical Ember app that uses observers?
