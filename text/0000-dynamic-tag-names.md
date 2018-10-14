- Start Date: 2018-10-14
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Dynamic tag names in glimmer templates.

## Summary

With the transition from inner-html semantics to outer-html semantics in components, we lost one feature: Being
able dynamically define the tag name of components.

I think it was an useful feature and we should find a way to bring it back.

## Motivation

Although not something we use every day, there is a need for some components to have a dynamic tag name.

This is most often used for certain low-level _presentational_ components.

Take for instance a component named `<Panel>` that is used to encapsulate some presentation concerns.
The template of that component could be like this:

```hbs
<div class="pt-10 pb-10 ps-20 box-shadow" ...attributes>
  {{yield}}
</div>
```

For accessibility and semantic reasons, sometimes a `<div>` may not be the best kind of tag.
We might want the panel to be a `<section>` element, or an `<aside>` when it's content is somewhat unrelated
with the rest of the page. Or maybe a `<legend>` if it's and the end of a form.

## Detailed design

The syntax I propose is to allow to interpolate just after the `<` and `</` markers in templates.

For instance:

```hbs
<{{@tagName}} class="pt-10 pb-10 ps-20 box-shadow" ...attributes>
  {{yield}}
</{{@tagName}}>
```

The glimmer parser needs to be able to distinguish this from any other interpolation in the so called
_element space_, like the future [Element Modifiers](https://github.com/emberjs/rfcs/pull/353).

This syntax presents two edge cases that we need clarification:

1 - What happens when the value in the interpolated in the opening and closing tags are different?

```hbs
<{{@tagName}} class="pt-10 pb-10 ps-20 box-shadow" ...attributes>
  {{yield}}
</{{@otherTagName}}>
```

This is clearly illegal and must throw an exception in development or production alike.

2 - What if the interpolated value is not a string? (`null`, `undefined` or a number/object).

Any value other than a String must throw an exception in development or production alike.

About `null`/`undefined` we can take two approaches.
We can deem them an error and throw an exception or we can take a similar approach than React did with `<>...</>`.
For those not familiar with `<></>` in React, that is used to generate DOM fragments. That is, the absence of a root
element.
I lean towards considering that a runtime error, but I could be convinced otherwise.


## How we teach this

This must be documented with examples in the guides, making very clear that the values being interpolated
have to be strings.

## Drawbacks

Admittedly this syntax is somewhat strange, and the fact that the exact same value has to be repeated in
both the opening and closing tags opens the door to making mistakes that should have good error messages.

## Alternatives

We can propose other alternative syntax. For instance, we could have a special built-in component for this:

```hbs
<DynamicElement @tagName={{@tagName}} class="some-class" ...attributes>
</DynamicElement>
```

## Unresolved questions

#### Should we allow helpers inside those interpolations?

For instance:
```hbs
<{{or @tagName "div"}}>

</{{or @tagName "div"}}>
```

I think that this repetition makes templates harder to read, so I advocate that only simple bindings
can be passed in. If the user wants to use something more complex than a binding, they could use the `let`/`with` helpers
for that.

```hbs
{{#with (or @tagName "div") as |tagName|}}
  <{{tagName}} class="foo">
    ...
  </{{tagName}}>
{{/with}}
```
