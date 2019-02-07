- Start Date: 2018-02-07
- Relevant Team(s): Ember.js, Ember Data
- RFC PR: https://github.com/emberjs/rfcs/pull/441
- Tracking: (leave this empty)

# Deprecate `computed().meta()`

## Summary

Deprecate the `meta` computed modifier.

## Motivation

Computed properties today have one last undeprecated modifier function, `meta`.
This function allows users to store some meta information with the computed
property to be looked up later on:

```js
let cp = computed(function() {
  // do some things
});

cp.meta({ someMetaInfo: 'someMetaValue' });

// later...

let meta = cp.meta();

console.log(meta); // { someMetaInfo: 'someMetaValue' })
```

This functionality was added a long time ago, before `WeakMaps` were commonly
available, and was used for passing meta information associated with a computed
around a larger system. The biggest user of this system is to this day Ember
Data, since they need to pass around lots of meta information about attributes
and relationships to be accessed later on.

This also being the last major modifier remaining, once it is removed we'll be
able to svelte _all_ of the modifiers, and remove them from production builds
of app that don't use them altogether.

## Transition path

The deprecation itself will be until Ember@v4, and will occur whenever anyone
uses the `meta` API or any associated APIs that expose the metadata, such as
`CoreObject#eachComputedProperty`, which currently accepts a callback that
receives the `meta` as the second parameter:

```ts
class CoreObject {
  eachComputedProperty(callback: (name: string, meta: any) => void): void;
}
```

The transition path will be to use `WeakMaps` to associate meta data with
computed property instances instead. APIs that currently expose meta information
directly, such as `eachComputedProperty`, will instead pass the original
computed instance so that users can grab associated meta information from their
`WeakMap`:

```ts
class CoreObject {
  eachComputedProperty(
    callback: (name: string, computed: ComputedDecorator) => void
  ): void;
}
```

This API and any others which expose computed properties for introspection like
this will still be considered private/intimate API, and should likely be removed
in future iterations since they expose internal implementation details of the
class, but the exact transition path for removing that is out of scope for this
RFC.

To handle the differences in these APIs while maintaining compatibility, addons
can use tools like [ember-compatibility-helpers](https://github.com/pzuraq/ember-compatibility-helpers/).

## How we teach this

This API was used mostly by a select few addons and has very limited/advanced
use cases, so this RFC proposes that we do not continue to document this exact
pattern. Instead, we should work with addon authors directly to figure out
better solutions for their use cases. The deprecation guide should still outline
a migration path, but should _avoid_ any usage of private/intimate APIs such as
`eachComputedProperty`.

## Drawbacks

This change will cause extra churn and maintenance burden on users of the `meta`
API, which is not ideal. There are relatively few users, and the work to update
will be relatively small, but this still represents a burden that those
maintainers will have to take on.
