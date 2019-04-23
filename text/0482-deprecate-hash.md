- Start Date: 2019-04-23
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/482
- Tracking: (leave this empty)

# Deprecate `hash` helper

## Summary

This RFC intends to rename the `hash` helper into `obj`.

## Motivation

The name of the helper can be confusing depending of the background of the developer.

JavaScript developers are not used to the hash terminology because they already have literal object notation like `let options = { url: 'http://www.example.com', method: 'GET' }`.

In addition, hash terminology can means different things depending on the context.
In another context, it can means cryptographic hash or a list of objects.

## Transition Path

The new `obj` helper should be implemented in Ember.js with the same behavior as `hash` helper which means it should returns an immutable object without prototype.
And this helper can't be override by a custom helper in your application.

Polyfill will helps to use more recent code without upgrading your version of Ember.
The codemod will helps to migrate deprecated code into stable code.
Since it's a strict renaming, there is no need to deprecate `hash` right away.

1. Implement `obj` helper in Ember.js
1. Polyfill `obj`
1. Write the codemod
1. Deprecate `hash`

## How We Teach This

The new `obj` helper maintains the same behavior as `hash`, so it is not necessary to explain it.

The term "object" and its short-name "obj" should be familiar to JavaScript developers since they are used frequently in code and when talking about code, for example when mentioning POJOs (Plain Old JavaScript Objects).

To use the new helper, you should change your code from:

```hbs
{{#let (hash name='Guillaume' job='Ember developer') as |person|}}
  <p>{{person.name}}</p>
{{/let}}
```

to:

```hbs
{{#let (obj name='Guillaume' job='Ember developer') as |person|}}
  <p>{{person.name}}</p>
{{/let}}
```

Alternatively, you can also run the codemod that will be provided.
This codemod should also be automatically run by `ember-cli-update` for the relevant versions.

For addon developers, you must use the polyfill as dependency to be compatible with all versions of Ember.

## Drawbacks

### Reserved name

If this RFC is implemented and you already declared a `obj` helper, it will throw an exception that you can't override the built-in helper.

## Alternatives

We could do nothing and leave things as is.

## Unresolved questions

### Obj or Object?

We could use the long name `object` for the helper.
This one is longer but more understandable.
