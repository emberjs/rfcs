- Start Date: 2020-02-20
- Relevant Team(s): Ember.js
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# `set` helper

## Summary

This RFC introduces a new `set` helper like the one provided by
[`ember-simple-set-helper`](https://github.com/pzuraq/ember-simple-set-helper).

The `set` helper will always return a function that will change a value.

```hbs
<SelectCountry @update={{set this.country}} />
```

The new `set` helper can be used instead of the old `{{fn (mut ..)}}` combination.
However this RFC does not deprecate `{{mut}}`.

## Motivation

The existing `mut` helper is quite helpful, but behaves often in an unexpected way.

When used to pass an argument as in `<MyComponent @foo={{mut this.bar}}>` or
`{{my-component foo=(mut this.bar)}}` it will do nothing for a glimmer component
but provide a box with a `value` and an `update` function on the private `attrs`
property of a classic component.

However the more common use-case is to pass the result of `mut` to either the 
`{{action}}` or the `{{fn}}` helper:

```hbs
<SelectCountry @update={{fn (mut this.country)}} />
<button {{on "click" (fn (mut this.name) "")}}>delete name</button>
```

However many people would expect `{{fn (mut this.foo)}}` to be the same as
`{{mut this.foo}}` in all cases, which can lead to confusion.

So this proposal proposes a new `set` helper that will work as `mut` wrapped
inside `fn`.

## Detailed design

The new `set` helper can be called with one or two arguments.

### one argument version

The one argument version returns a function that receives one argument `value` and
will set the property passed as argument to `value`.
```hbs
{{set context.property}}
```

```js
return function(arg) {
  context.property = arg;
};
```

The first argument must *always* have a context. So `{{set foo}}` or `{{set @foo}}` is not valid.
This aligns with [RFC 0308](https://github.com/emberjs/rfcs/blob/master/text/0308-deprecate-property-lookup-fallback.md).

### two argument version

The two argument version `{{set context.property value}}` is a shorthand for
`{{fn (set context.property) value}}` and will always perform the same.

## How we teach this

We can teach teach it the same way as we used to tech `mut`.
We must replace all occurances of `mut` in the guides and replace it by `set`.

## Drawbacks

It increases API surface, and provides two helpers for the same functionallity.
However this can be mitigated by clearly communicating that `set` should be
used in favor of `mut`.

## Alternatives

We could leave everything as it is and encourage people to use the `{{fn (mut` combination.
Or we could try to change `mut` to behave like the `set` proposed here with an *optional feature*.

## Unresolved questions

Do we want the two argument version? We could just go with the one argument version.