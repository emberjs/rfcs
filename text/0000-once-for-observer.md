- Start Date: 2018-12-28
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Add `once` flag to observer

## Summary

> `addListener` has a flag called `once`. This flag could be useful for observers as well.

## Motivation

> Adding the flag `once` to observers would improve the consistency between `addListener` and `addObserver`.

## Detailed design

> The `once` flag of `addListener` tells to call the method only once. This RFC aims to add this flag to observers as well.

## How we teach this

> The best would be to follow the terminology used in listeners because these are already known by Ember developers.

> Only the API should be rebuild after this RFC.

> This feature does not need big publicity. This should be only mentioned in the release note.

## Drawbacks

> Observers are not very popular in EmberJS.

> There are no really tradeoffs except that observers would get a bit publicity.

## Alternatives

> The addon `ember-observer-macros` can be used. It has `observerOnce`.

## Unresolved questions
