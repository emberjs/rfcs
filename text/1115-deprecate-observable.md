---
stage: accepted
start-date: 2025-06-16T00:00:00.000Z
release-date:
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1115
project-link:
---

# Deprecate Observers

## Summary

Deprecate the `Observable` Mixin, `observer` helper, and public observer functionality.

## Motivation

Using observers has been soft-deprecated ever since the introduction of tracking. Removing
support for them will significantly clean up the code base. (Note that for us to entirely
remove observer support internally, we will likely need to deprecate other functionality
in addition to what is include here.)

Additionally, Ember has also not recommended the use of Mixins for a while. Since
`Observable` uses the Mixin architecture, it will also need to be deprecated for us
to fully remove Mixins.

## Transition Path

### Observable Mixin

Some of the methods provided by `Observable` are available as standalone methods,
for these the transition path is straight-forward:

* `get` -> Import from `@ember/object`
* `getProperties` -> Import from `@ember/object`
* `set` -> Import from `@ember/object`
* `setProperties` -> Import from `@ember/object`
* `notifyPropertyChange` -> Import from `@ember/object`

For sugar helpers they can easily be replaced with a bit more verbosity:

* `incrementProperty`
    ```js
    this.count += 1; // Tracking
    set(this, 'count', get(this, 'count') + 1); // Legacy 
    ```
* `decrementProperty`
    ```js
    this.count -= 1; // Tracking
    set(this, 'count', get(this, 'count') - 1); // Legacy 
    ```
* `toggleProperty`
    ```js
    this.flag = !this.flag; // Tracking
    set(this, 'flag', !get(this, 'flag')); // Legacy
    ```

For others, there will no replacement and code that depends on them
will have to be refactored:

* `addObserver`
* `removeObserver`
* `cacheFor`

### Observable Helper

This function does not work with native class syntax and as such, shouldn't be relied upon.
It will not be replaced.

## Exploration

To validate this deprecation, I've tried removing this functionality in the following PRs:
* Inline Observable Mixin: https://github.com/emberjs/ember.js/pull/20928
* Remove `observer` helper and inlined Observable methods: https://github.com/emberjs/ember.js/pull/20937

## How We Teach This

We should remove all references to this functionality from the guides.

## Drawbacks

Many users probably rely on this functionality. However, it's almost certainly
something that we don't want to support long-term.

## Alternatives

* Just inline this functionality so that we can at least get rid of Mixins.

## Unresolved questions

None