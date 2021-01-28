---
Stage: Accepted
Start Date: 2019-12-08
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/561
---

# Adding Numeric Comparison Operators to Templates

## Summary

Add new built-in template `{{lt}}` and `{{gt}}` helpers to perform basic numeric comparison operations in templates, similar to those included in `ember-truth-helpers`.

This RFC is a subset of the changes proposed in #388.

## Motivation

It is a very common need in any sufficiently complex Ember app to perform some numeric comparison operations and often the most convenient place to do it is right in the templates.
Because of that, [ember-truth-helpers](https://github.com/jmurphyau/ember-truth-helpers) is one of the most installed addons out there, either directly by apps or indirectly by
other addons that those apps consume.

The fact that `ember-truth-helpers` is so popular is a good signal that this it is filling a perceived gap in Ember's functionality.

A second reason is that it might help make Ember more approachable to newcomers that have some experience in other frameworks.
Most if not all web frameworks have some way of comparing numbers in the templates and it's surprising that Ember requires an third party package to perform
even the most basic operations.


## Detailed design

Add `{{lt}}` and `{{gt}}` helpers.

#### `{{lt}}`
Binary operation. Throws an error if not called with exactly two arguments.
Equivalent of <arg1> < <arg2>
This is identical to the `{{lt}}` helper in `ember-truth-helpers`

#### `{{gt}}`
Binary operation. Throws an error if not called with exactly two arguments.
Equivalent of <arg1> > <arg2>
This is identical to the `{{gt}}` helper in `ember-truth-helpers`, except for the name.

This RFC intentionally leaves the implementation details unspecified, those could be implemented in Glimmer VM or
in a higher level in Ember itself.

## How we teach this

The introduction of these helpers does not impact the current mental model for Ember applications.

In addition to API and Guides documentation with illustrative examples of usage of the various helpers.

## Drawbacks

Adding new helpers increases the surface area of the framework and the code the core team commits to support long term.

## Alternatives

One alternative path is don't do anything and let users continue to define their own helpers (or install `ember-truth-helpers`).

## Unresolved questions

- The proposed version of those helpers mimic the behavior of the `<` and `>` operators in Javascript. While this is the
  least surprising thing to do, one could argue that we could add extra logic to protect users from the usual pitfalls
  and edge cases of those operators, **like throwing exceptions if any of the values is `NaN` or a value other than a number**.
- We must decide if it's worth adding `{{lte}}` and `{{gte}}` helpers, equivalent to `<=` and `>=` respectively.
- `BigInt` support. The BigInt proposal has recently reached stage 4 and will be part of ECMASCRIPT 2020. According to
  the spec comparisons using `<` and `>` work as expected and I don't anticipate problems, but ensure we explicitly
  test for those numbers.
