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

### Hacky "solution"
If the value is not a variable `{{}}` can actually be ommited as a `true` string is truthy leading to somewhat easier to remember syntax of
```hbs
<MyHackyBoolArgument @bool=true />
```
Since this is hacky, one have to remember that
```hbs
<MyHackyBoolArgument @bool=false />
```
would not work as `@bool=false` would pass a `false` as string that is truthy in JavaScript.

For simple usecases  it is verbose to write and does not align with HTML attributes like `disabled` of which value can be ommited as desribed in [HTML Spec](https://www.w3.org/TR/html5/infrastructure.html#sec-boolean-attributes)

## Detailed design

This RFC proposes that if an argument is passed without any value it should have a value `true` set inside of a component.

e.g.
```hbs
<Button @secondary />
```
inside the Button component `@secondary` should equal to `true`.
e.g.
```hbs
<button class={{if @secondary "is-secondary"}}>
  Button
</button>
```

should render as
```html
<button class="is-secondary">
  Button
</button>
```

### Align with attributes
This would align with how attributes behave currently.
Given an Input component:
```hbs
<input ...attributes>
```

if it is invoked like this:
```hbs
<Input disabled />
```

it will render as
```html
<input disabled>
```


## How we teach this

This should be included in guides and also should be easier to teach as it is inline with HTML spec for attributes.

## Drawbacks

Can't think of any at the moment.

## Alternatives

- In both React and Vue if a prop is passed with no value it is truthy inside of a component.
  - https://vuejs.org/v2/guide/components-props.html#Passing-a-Boolean
  - https://reactjs.org/docs/jsx-in-depth.html#props-default-to-true

## Unresolved questions

None at the moment.
