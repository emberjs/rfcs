- Start Date: 2020-02-20
- Relevant Team(s): Ember.js
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# `set` helper

## Summary

This RFC introduces a new `set` helper like the one provided by
[`ember-simple-set-helper`](https://github.com/pzuraq/ember-simple-set-helper).

The `set` helper will always return a function that will change value.

```hbs
<SelectCountry @update={{set this.country}} />
```

The new `set` helper can be used instead of the old `{{fn (mut ..)}}` combination.
However this RFC does not deprecate `{{mut}}`.
It's unclear if the `<Component @data={{mut this.data}}>` syntax still has a use case,
and if we need a solution for dynamic attribute setting before deprecating `mut`.
So future RFCs will address this and then deprecate `mut`.

## Motivation

The existing `mut` helper is quite helpful, but behaves often in an unexpected way.

### `mut` is a dualistic helper

The primary problem is that `mut` basically has two totally different behaviors.

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

### The details of the problems with `mut`

The problem is that while people expect `mut` to return a *setter function* it will actually
return some kind of *mut box*. The *mut box* behaves differently depending how it is used.

When a *mut box* is passed to `fn` (or `action`) it will return a *setter function*, which is
what people expect. However when a *mut box* is passed to a component with `@data={{mut ...}}`
it will set the *value* to `this.args.data` for glimmer and `this.data` for classic components.
However `@data` will be the *mut box* itself. On classic components the *mut box* is also exposed on `this.attrs.data` which is private API.

The problem is that often people expect `mut` to return a *setter function* while it is just
a *mut box* because they dont know that `fn` converts a *mut box* to a *setter function*.

So lets assume a `update` action is passed with `mut` but without `fn`:

```hbs
<SelectCountry @update={{mut this.country}} />
```

```hbs
{{#each this.countries as |country|}}
  <button {{on "click" (fn @update country)}}>{{country}}</button>
{{/each}}
```

This works but now the call to `<SelectCountry>` depends on an implementation detail of the
`<SelectCountry>` component. It does only work because the `fn` helper is used to call
`@update`, and so here the *mut box* is converted to a *setter function*.

When `<SelectCountry>` is refactored to call `update` from `js` it will break:

```hbs
{{#each this.countries as |country|}}
  <button {{on "click" (fn this.selectCountry country)}}>{{country}}</button>
{{/each}}
```

```js
@action selectCountry(country) {
  this.args.update(country); // this breaks
}
```

It breaks because `this.args.update` will be the *value* of `country`,
not its *setter function*.

It will work when the call to `<SelectCountry>` is fixed to actually pass
a *setter function*:

```hbs
<SelectCountry @update={{fn (mut this.country)}} />
```

Or when `<SelectCountry>` is a classic component it could use `this.attrs` to
access the private *mut box*:

```js
@action selectCountry(country) {
  this.attrs.update.update(country);
}
```

### a new `set` helper that always returns a *setter function*

This proposal proposes a new `set` helper with the semantics users expect from `mut`:
the way it behaves when wrapped with the `fn` helper.

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

The fact that the `set` helper somehow needs to know the *context* and the *name* of the
argument it receives makes it magic syntax that can not be used by a userland helper without
a AST transformation. But the need for the helper seems big enough to justify this, especially
when the same magic syntax is already used by the `mut` helper.

### two arguments version

The two-argument version `{{set context.property value}}` is a shorthand for
`{{fn (set context.property) value}}` and will always perform the same.

### dynamic attribute binding

The existing `mut` helper allows to receive the result of the `get` helper to create a 
[dynamic attribute binding](https://guides.emberjs.com/release/components/built-in-components/#toc_binding-dynamic-attribute):

```hbs
<Input @value={{mut (get this.person this.field)}}
```

This use-case is *not* covered by the new `set` helper, however the existing `mut` helper
can still be used for this.

## How we teach this

We can teach it the same way as we used to teach `mut`.
We must replace all occurrences of `mut` in the guides and replace them by `set`.

## Drawbacks

It increases the API surface and provides two helpers for the same functionality.
However this can be mitigated by clearly communicating that `set` should be
used in favor of `mut`.

## Alternatives

We could leave everything as it is and encourage people to use the `{{fn (mut` combination.
Or we could try to change `mut` to behave like the `set` proposed here with an *optional feature*.

One option is to not introduce magic syntax and make a `{{set context "propertyName"}}`helper.
This one would also work dynamically as in `{{set context dynamicPropertyName}}`, but it makes the property name a string, which makes it harder to spot that a value and a setter is passed:

```hbs
when we use a magic syntax:
<SelectCountry @value={{this.country}} @update={{set this.country}} />
without magic syntax:
<SelectCountry @value={{this.country}} @update={{set this "country"}} />
```

## Unresolved questions

Do we want to introduce the magic syntax?
It seems reasonable to have a special syntax to set a value, but we could also go
with the string version.

And do we want the two arguments version? We could just go with the one argument version.
One reason to *not* do a two argument version is that we could later in a second RFC
could use the second argument to set dynamic properties:

```hbs
{{#each (array "origin" "destination") as |property|}}
  <SelectCountry @update={{set this property}}>
{{/each}}
```
