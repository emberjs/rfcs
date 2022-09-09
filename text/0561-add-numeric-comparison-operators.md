---
stage: accepted
start-date: 2019-12-08T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/561
project-link:
---

# Adding Comparison Operators to Templates

## Summary

Add new built-in template `{{lt}}`, `{{lte}}`, `{{gt}}`, and `{{gte}}` keywords to perform basic numeric comparison operations in templates, similar to those included in `ember-truth-helpers`.

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

The new comparison operators will be made keywords, so they are easily accessible in current and future templates without needing to be imported.

#### `{{lt}}`

Binary operation. Throws an error if not called with exactly two arguments.
Equivalent of <arg1> < <arg2>
This is identical to the `{{lt}}` helper in `ember-truth-helpers`

#### `{{lte}}`

Binary operation. Throws an error if not called with exactly two arguments.
Equivalent of <arg1> <= <arg2>
This is identical to the `{{lte}}` helper in `ember-truth-helpers`

#### `{{gt}}`

Binary operation. Throws an error if not called with exactly two arguments.
Equivalent of `<arg1> > <arg2>`
This is identical to the `{{gt}}` helper in `ember-truth-helpers`.

#### `{{gte}}`

Binary operation. Throws an error if not called with exactly two arguments.
Equivalent of `<arg1> >= <arg2>`.
This is identical to the `{{gte}}` helper in `ember-truth-helpers`.

## How we teach this

While the introduction of these helpers doesn't introduce new concepts, as helpers like these could be written and in fact were written for a long time, it might affect slightly how we frame some concepts in the guides.

Previously users were encouraged to put computed properties in the JavaScript file of the components, even for the most simple tasks like comparing if a value is less than another using `computed.lt`.

With the addition of these helpers users don't have to resort to computed properties for simple operations, which sometimes forced users to create JavaScript files for what could have been template-only components.

In addition to documenting the new helpers in the API docs, the Guides should be updated to favour the usage of helpers over computed properties where it makes more sense, adding illustrative examples and stressing out where the definition of truthiness of handlebars differs from the one of Javascript.

### API Docs

#### `{{lt}}`

The `{{lt}}` helper can be used to compare two values in a template. It returns `true` if the first value is
less than the second value, and `false` otherwise. It is equivalent to the `<` operator in JavaScript.

```hbs
{{#if (lt @number 5)}}
  The number is less than 5!
{{/if}}
```

#### `{{lte}}`

The `{{lte}}` helper can be used to compare two values in a template. It returns `true` if the first value is
less than or equal to the second value, and `false` otherwise. It is equivalent to the `<=` operator in JavaScript.

```hbs
{{#if (lte @number 5)}}
  The number is less than or equal to 5!
{{/if}}
```

#### `{{gt}}`

The `{{gt}}` helper can be used to compare two values in a template. It returns `true` if the first value is
greater than the second value, and `false` otherwise. It is equivalent to the `>` operator in JavaScript.

```hbs
{{#if (gt @number 5)}}
  The number is greater than 5!
{{/if}}
```

#### `{{gte}}`

The `{{gte}}` helper can be used to compare two values in a template. It returns `true` if the first value is
greater than or equal the second value, and `false` otherwise. It is equivalent to the `>=` operator in JavaScript.

```hbs
{{#if (gte @number 5)}}
  The number is greater than or equal to 5!
{{/if}}
```

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
