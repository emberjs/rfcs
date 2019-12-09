- Start Date: 2019-12-08
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: https://github.com/emberjs/rfcs/pull/562
- Tracking: (leave this empty)

# Adding Logical Operators to Templates

## Summary

Add new built-in template `{{and}}`, `{{or}}` and `{{not}}` helpers to perform basic logical operations in templates, similar to those included in `ember-truth-helpers`.

This RFC is a subset of the changes proposed in #388.

## Motivation

It is a very common need in any sufficiently complex Ember app to perform some logical comparison operations and often the most convenient place to do it is right in the templates.
Because of that, [ember-truth-helpers](https://github.com/jmurphyau/ember-truth-helpers) is one of the most installed addons out there, either directly by apps or indirectly by
other addons that those apps consume.

The fact that `ember-truth-helpers` is so popular is a good signal that this it is filling a perceived gap in Ember's functionality.

A second reason is that it might help make Ember more approachable to newcomers that have some experience in other frameworks.
Most if not all web frameworks have some way of performing logical operations in the templates and it's surprising that Ember requires an third party package to perform
even the most basic operations.

A third reason, this time technical, is that by implementing those helpers at a lower-level, we can make them more performant by making them short-circuit.
Right now helpers implemented using public APIs like those in `ember-truth-helpers` eagerly consume their arguments. In the case of logical operations like `{{and a b}}`,
once the first argument (`a`) is evalued to _falsey_, the second argument (`b`) is irrelevant. Sometimes arguments can be expensive properties to calculate,
and by short-circuiting we can avoid computing them at all sometimes.


## Detailed design

Add `{{and}}`, `{{or}}` and `{{not}}` helpers.

#### `{{and}}`
Takes at least two positional arguments, bound or unbound. Raises an error if invoked with less than two arguments.
It evaluates arguments left to right, returning the first one that is not _truthy_ (**by handlebar's definition of truthiness**)
or the right-most arguments if **all** evaluate to _truthy_.
This is *NOT* equivalent to the `{{and}}` helper from `ember-truth-helpers` because unlike this proposed helper, the one in `ember-truth-helpers`
uses Javascript's definition of truthiness.

#### `{{or}}`
Takes at least two positional arguments, bound or unbound. Raises an error if invoked with less than two arguments.
It evaluates arguments left to right, returning the first one that is _truthy_ (**by handlebar's definition of truthiness**) or the
right-most argument if all evaluate to _falsy_.
This is *NOT* equivalent to the `{{or}}` helper from `ember-truth-helpers` because unlike this proposed helper, the one in `ember-truth-helpers`
uses Javascript's definition of truthiness.

#### `{{not}}`
Unary operator. Raises an error if invoked with more than one positional argument. If the given value evaluates to a _truthy_ value (**by handlebar's definition of truthiness**),
the `false` is returned. If the given value evaluates to a _falsy_ value (**by handlebar's definition of truthiness**) then it returns `true`.
This is *NOT* equivalent to the `{{not}}` helper from `ember-truth-helpers` because unlike this proposed helper, the one in `ember-truth-helpers`
uses Javascript's definition of truthiness.

#### Handlebar's definition of truthiness
This is the most important detail of this proposal because it's where it deviates from `ember-truth-helpers`.
Handlebars has its own definition of _truthyness_, which is similar to Javascripts except that empty arrays are
considered **falsy**, while in JS are considered **truthy**.


This RFC intentionally leaves the implementation details unspecified, but one can think of those helpers as macros that
expand to combinations of `if`s.

##### `{{and}}`
- `{{and a b}}` is equivalent to `{{if a b a}}`
- `{{and a b c}}` is equivalent to  `{{if a (if b c b) a}}`
- and so on

##### `{{or}}`
- `{{or a b}}` is equivalent to `{{if a a b}}`
- `{{and a b c}}` is equivalent to  `{{if a a (if b b c)}}`
- and so on

##### `{{not}}`
- `{{not a}}` is equivalent to `{{if a false true}}`

## How we teach this

The introduction of these helpers does not impact the current mental model for Ember applications.

In addition to API and Guides documentation with illustrative examples of usage of the various helpers, stressing out
where the definition of truthiness of handlebars is different from the one in Javascript.

## Drawbacks

Adding new helpers increases the surface area of the framework and the code the core team commits to support long term.

## Alternatives

One alternative path is to not take any action and let users continue to define their own helpers (or install `ember-truth-helpers`).

## Unresolved questions

- Consider following Javascript's definition of truthiness. This would also help with the transition from `ember-truth-helpers`.
