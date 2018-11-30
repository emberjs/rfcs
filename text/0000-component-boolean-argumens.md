- Start Date: 11/30/2018
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Component boolean Arguments

## Summary

This RFC proposes to add a boolean argument to components, that aligns with boolean attribute of regular DOM elements.

## Motivation

Currently to pass a boolean property to a component a developer has to explicitly pass `true` as an argument.
e.g.
```hbs
<Button @secondary={{true}} />
```
This does allow the value to be a variable e.g.

```hbs
<Dialog @visible={{this.visible}} />
```

If the value is not a variable `{{}}` can actually be ommited as a `true` string is truthy leading to somewhat easier to remember syntax of
```hbs
<MyHackyBoolArgument @bool=true />
```

For simple usecases  it is verbose to write and does not align with HTML attributes like `disabled` of which value can be ommited as desribed in [HTML Spec](https://www.w3.org/TR/html5/infrastructure.html#sec-boolean-attributes)

## Detailed design

This RFC proposes that if argument is passed without any value it should have value `true` set inside of a component.

```hbs
<Form @disabled />
```
inside the Form component `@disabled` would equal to `true`.


## How we teach this

This should be included in guides and also should be easier to teach as it is inline with HTML spec for attributes.

## Drawbacks

Can't think of any at the moment.

## Alternatives

- In React if a prop is passed with no value it is truthy inside of a component.
- In Vue if a prop is passed with no value it is falsy inside of a component.

## Unresolved questions

None at the moment.
