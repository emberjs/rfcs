- Start Date: 2018-10-13
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Promote helpers from ember-truth-helpers into Ember

## Summary

Add new built-in template helpers to perform basic operations in templates, identical to the
helpers in [ember-truth-helpers](https://github.com/jmurphyau/ember-truth-helpers)

## Motivation

It is a very common necessity to almost every Ember app to perform certain operations like compare
by equality, negate a value, or perform boolean operations, and often the most convenient place to
do it is in the templates.
Because of that, `ember-truth-helpers` is probably the single most installed addon either directly
or transitively in the entire ecosystem.

The fact that this addon is so popular is very telling and Ember should consider to move into the
core of the templating engine at least the most popular helpers.

One of the shocking moments developers that are familiar with React, Vue or Angular have when trying Ember
is that they cannot, at least out of the box, perform the most basic comparisons and boolean operations
they are so used to in JSX or Vue/Angular templates.

Furthermore, if the [RFC #367](https://github.com/emberjs/rfcs/pull/367) eventually gets merged, these
helpers are the perfect candidates to create adoption friction in the community because of how
pervasive its usage it in so many templates.

Albeit less important, implementing these helpers in the core _could_ enable clever trickery in the
Glimmer VM, like evaluating them at compile time when the arguments are constant.

## Detailed design

The process consists on deciding what helpers from `ember-truth-helpers` we consider the most important
and move them into Ember.js itself or even the Glimmer VM, in a fully backwards compatible way.

I propose to move to core at least:

- `eq`
- `not`
- `and`
- `or`
- `gt` and `gte`
- `lt` and `lte`

Those helpers are very low lever and generally useful in both Ember and Glimmer.

I propose to not move:

- `is-array` (uses `Ember.isArray`)
- `is-empty` (uses `Ember.isEmpty`)
- `is-equal` (uses `Ember.isEqual`)
- `xor`      (not very common)
- `not-eq`   (can be expressed as `(not (eq a b)`)

When this feature is implemented, we would update `ember-truth-helpers` to automatically remove
the promoted helpers from the code when the Ember is above a certain version number.

## How we teach this

As any new API, will have to be documented in the guides with examples.

## Drawbacks

We are increasing the public API of the framework, and every line of code is a liability, although
those helpers are extremely straightforward.

## Alternatives

An alternative path would be to include `ember-truth-helpers` in the default blueprint for apps an
addons.

## Unresolved questions

The main unresolved question is what helpers we deem worthy of being moved into the core, and also
if we want any other helpers not mentioned above.

Other helpers that come to mind that _could_ also be worth adding:

- `array` (there is an RFC for it: https://github.com/emberjs/rfcs/pull/318)
- `add`, `subtract`, and other arithmetic operators.
- `{{#await promise as |value|}}` to render a block conditionally if a promise resolves or fails.

If there is interest in them, I advocate creating standalone RFCS for them.