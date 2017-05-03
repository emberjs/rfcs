- Start Date: 2015-01-18
- RFC PR:
- Ember Issue:

# Summary

Allow HTMLBars components to use `number` and `boolean` literals for attribute
values.

# Motivation

When using the new component syntax introduced in the
`ember-htmlbars-component-generation` feature, it's not possible to pass
literals other than strings to an attribute. With the current mustache component
syntax this works fine: `num`, `bool`, `str` get converted to `number`,
`boolean` and `string`, respectively:

```hbs
{{x-foo num=100 bool=false str="bar"}}
```

The new component syntax, however, only support `string` type, even though the
values are unquoted.

```hbs
<x-foo num=100 bool=false str="bar"></x-foo>
```

The behavior of Handlebars is very convenient, most of the projects rely on
passing a number for a custom image component's `width`/`height` attribute, or
to configure the behavior of a component using booleans. Having to manually
convert attribute values to their desired types introduce an unnecessary and
very tedious an extra step to the process.

# Detailed design

Currently HTMLBars doesn't seem to distinguish between quoted or unquoted
atrributes at parse time. This template (with `disableComponentGeneration`
set to `false`):

```html
<x-foo bool=true num=100 />
```

compiles to the following code:

```js
(function() {
  // *snip*
  return {
    // *snip*
    render: function render(context, env, contextualElement) {
      // *snip*
      component(env, morph0, context, "x-foo", {"bool": "true", "num": "100"}, child0);
      return fragment;
    }
  };
}())
```

Basically it just passes the string values of the `bool` and `num` attributes to
the `component` function. At this point, Ember has no way to figure out the
original intention. If HTMLBars emitted literals for booleans and numbers, it
would make the problem solvable.


# Drawbacks

Not sure if any.


# Alternatives

For the sake of completeness, below is how the other prominent frameworks handle
the case.


### Polymer

Polymer is interesting, because if a default value is provided, it'll try to
convert the attr's type to match the default value's type. The problem is, if
the default value is `null`, Polymer will not bother with the conversion, which
result the value to always be string. As there are cases where an initial `null`
value is perfectly valid, this behavior could cause a lot of confusion.

http://jsbin.com/zeqira/1/edit?html,console


### Angular

Angular uses `string` type if the attr is accessed through the `attrs` argument
of the linking function. It will, however, try to convert the type of the
attribute's value to `number` or `boolean` if the attribute is accessed through
the `scope` arg, *regardless the attribute is quoted or not*.

http://jsbin.com/wireqa/1/edit?html,console


### React

React's JSX compiler is probably the most straightforward, as it warns the user
to either quote the attribute (`string`), or use curlies (literal).

http://jsbin.com/bugive/1/edit?html,console


# Unresolved questions

Nothing that I know of, but surely some will pop up.
