---
Stage: Accepted
Start Date: 2020-12-22
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/691
---

# Deprecate passing `classBinding` and `classNameBindings` as arguments

## Summary

In Ember today, it is possible to pass `classBinding` and `classNameBindings` as
arguments to a component when invoked with curly syntax.

```hbs
{{some-component classNameBindings="foo:truthy:falsy"}}
```

These arguments are merged into the `class` attribute on the class, regardless
of whether or not the component is a classic component which contains the
`classNameBindings` logic. It is also fully possible to accomplish with template
syntax in alternative ways, so this RFC proposes deprecating them.

## Motivation

`classBinding` and `classNameBindings` are Classic component APIs for
manipulating the class name of the element that wraps the component. These were
"merged properties" on classic components historically, which meant that rather
than overwriting the values, they would merge into the superclass when extending
a class. And for _instances_ of components this was also true, so passing it as
an argument into a component instance would _add_ the name bindings to the
class, rather than replacing them.

At some point, the ability for merging on the instance itself was removed. In
its place, a general template transform was added, which transformed the
`classBinding` and `classNameBindings` args into the `class` argument, with the
same semantics.

```hbs
{{some-component class="bar" classNameBindings="foo:truthy:falsy"}}

{{! becomes }}
{{some-component class=(concat "bar " (if this.foo "truthy" "falsy"))}}
```

Note the conversion of the `foo` string into a path in the local context,
something that is very much unexpected in modern Ember applications.

The output here is a bit cumbersome to write, which may be why the original
syntax was kept. However, with angle bracket invocation, we can now write this
in a much cleaner way:

```hbs
<SomeComponent class="bar {{if this.foo "truthy" "falsy"}}" />
```

Given these alternatives, it doesn't make sense any longer to continue
supporting this transform, _especially_ given that it also affects components
which do not even support `classNameBindings`, such as Glimmer components. Since
`class` in this case is an _argument_ and not an attribute, and since this only
affects curly components, it is unlikely that anyone is actually relying on this
behavior, but removing this incongruity seems like the best course of action.

## Transition Path

In general, users should convert to use angle bracket syntax to invoke their
component, and then use standard interpolation within the `class` attribute of
the component. In cases where this is not possible/desired, users can continue
using curly invocation with `(concat)` to pass the value.

## How We Teach This

### Deprecation Guide

`classBinding` and `classNameBindings` can currently be passed as arguments to
components that are invoked with curly invocation. These allow users to
conditionally bind values to the `class` argument using a microsyntax similar to
the one that can be defined in a Classic component's class body:

```js
import Component from '@ember/component';

export default Component.extend({
  classNameBindings: ['isValid:is-valid:is-invalid']
});
```

```hbs
{{my-component classNameBindings="isValid:is-valid:is-invalid"}}
```

Each binding is a triplet separated by colons. The first identifier in the
triplet is the value that the class name should be bound to, the second
identifier is the name of the class to add if the bound value is truthy, and the
third value is the name to bind if the value is falsy.

These bindings are additive - they add to the existing bindings that are on the
class, rather than replacing them. Multiple bindings can also be passed in by
separating them with a space:

```hbs
{{my-component
  classBinding="foo:bar"
  classNameBindings="some.boundProperty isValid:is-valid:is-invalid"
}}
```


These bindings can be converted into passing a concatenated string into the
class argument of the component, using inline `if` to reproduce the same
behavior. This is most conveniently done by converting the component to use
angle-bracket invocation at the same time.

Before:

```hbs
{{my-component
  classBinding="foo:bar"
  classNameBindings="some.boundProperty isValid:is-valid:is-invalid"
}}
```

After:

```hbs
<MyComponent
  class="
    {{if this.foo "bar"}}
    {{if this.some.boundProperty "bound-property"}}
    {{if this.isValid "is-valid" "is-invalid"}}
  "
>
```

Note that we are passing in the `class` attribute, not the `class` argument. In
most cases, this should work exactly the same as previously. If you referenced
the `class` argument inside of your component, however, you will need to pass
`@class` instead.

If you do not want to convert to angle bracket syntax for some reason, the same
thing can be accomplished with the `(concat)` helper in curly invocation.

```hbs
{{my-component
  class=(concat
    (if this.foo "bar")
    " "
    (if this.some.boundProperty "bound-property")
    " "
    (if this.isValid "is-valid" "is-invalid")
  )
}}
```

## Drawbacks

- Introduces a minor amount of churn.
