- Start Date: 2018-10-13
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Add new basic template helpers to Ember

## Summary

Add new built-in template helpers to perform basic boolean operations and comparisons in templates, identical to some of the
helpers in [ember-truth-helpers](https://github.com/jmurphyau/ember-truth-helpers).

This is a resurrection of [RFC #152](https://github.com/emberjs/rfcs/pull/152) that was opened over
by @martndemus two years ago, updated to reflect the new start of the things in Ember, with the glimmer
VM powering our templates.

## Motivation

It is a very common necessity to almost every Ember app to perform certain operations like compare
by equality, negate a value or perform boolean operations, and often the most convenient place to
do it is right in the templates.
Because of that, `ember-truth-helpers` is probably the single most installed addon that exists, either
directly by apps or indirectly by other addons that those apps use.

The fact that this addon is so popular is very telling and Ember.js should consider moving into the
core of the templating engine at least some of those helpers.

A second reason is that I believe it would help making Ember more approachable by newcomers
that have _some_ experience in other frameworks. One of the most shocking moments that developers
that are familiar with React, Vue or Angular experience when trying Ember is that they cannot,
at least out of the box, perform the most basic logical comparisons and operation they are so used to
in JSX or Vue/Angular templates.

A third reason is that if we implement those super common helpers in the Glimmer VM, that would open
a important vector of low level optimization.

Consider the following template:

```hbs
{{#if (and @featureEnabled this.expensiveComputedProperty @model.asyncEDRelationship.length)}}
  {{!-- some logic --}}
{{else}}
  {{!-- some other logic --}}
{{/if}}
```

Because of the way Ember helpers work, all their input parameters are eagerly evaluated by the
Glimmer VM and passed to the helpers. This might include computationally expensive computed properties,
or hitting code paths that trigger network requests.

Implementing helpers like `and` or `or` at a lower lever would allow those helpers to be evaluated in
short circuit, so if `@featureEnabled` is false, neither of the following arguments will ever evaluated.

Going one step further into the optimizing compiler world, if **every** invocation of the component
with this template receives `@featureEnabled={{false}}`, the compiler could even completely remove
the conditional and the truthy branch of the if from compiled output.

The last reason is that if the [RFC #367](https://github.com/emberjs/rfcs/pull/367) eventually gets merged,
these helpers are the perfect candidates to create adoption friction in the community because of how
pervasive its usage it in so many templates, forcing them to explicitly importing them on most templates
or adding them to the proposed `prelude.hbs` file.

## Detailed design

The process consists on deciding what helpers from `ember-truth-helpers` we consider the most important
and move them into Ember.js itself or even the Glimmer VM, in a fully backwards compatible way.

I propose to add to core at least:

- `eq`
- `not`
- `and`
- `or`
- `gt` and `gte`
- `lt` and `lte`

Those helpers are very low lever and generally useful in both Ember and Glimmer.

I propose to not add:

- `is-array` (uses `Ember.isArray`)
- `is-empty` (uses `Ember.isEmpty`)
- `is-equal` (uses `Ember.isEqual`)
- `xor`      (not very common)
- `not-eq`   (can be expressed as `(not (eq a b)`)

When this feature is implemented, we would update `ember-truth-helpers` to automatically remove
the promoted helpers from the code when the Ember is above a certain version number.

## How we teach this

The introduction of these helpers does not impact the current mental model for Ember applications.

In addition to API and Guides documentation with illustrative examples of usage of the various helpers,
and explanation of their short circuiting nature might be warranted.

## Drawbacks

We are increasing the public API of the framework, and every line of code is a liability, although
those helpers are extremely straightforward.

## Alternatives

An alternative path would be to include `ember-truth-helpers` in the default blueprint for apps and
addons.
However, this alternative loses strength due to the fact that it is not possible to implement short
circuiting helpers in Ember's userspace, and because even if an addon is added by default, user
can still choose to remove it, so addons would not just be able to rely on them.

## Unresolved questions

The main unresolved question is what helpers we deem worthy of being moved into the core, and also
if we want any other helpers not mentioned above.

In particular, I'm sitting on the fence about `not-eq`.
It is not _really necessary_, in the same way `{{unless foo}}` can also be expressed as `{{#if (not foo)}}`,
but it may be convenient.

Other helpers that come to mind that _could_ also be worth adding:

- `array` (there is an [RFC](https://github.com/emberjs/rfcs/pull/318) and its being implemented)
- `add`, `subtract`, and other arithmetic operators.
- `{{#await promise as |value|}}` to render a block conditionally if a promise resolves or fails.

If there is interest in them, I advocate creating standalone RFCS for each one of them.
