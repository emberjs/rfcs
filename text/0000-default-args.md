---
Stage: Accepted
Start Date: 2020-12-26
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR:
---

# Argument Default Primitives

## Summary

Add a `getDefaultArgs` hook to component managers which can be used to provide
default arguments to components.

## Motivation

With the introduction of Glimmer components and Template-only components which
do not have an implicit backing class, Ember users no longer have a way to
provide default arguments to components. Previously, they could do this by
assigning a value to the property with the same name on the component class, and
refer to the argument directly on the class instance:

```js
// app/components/my-component.js
import Component from '@ember/component';

export default class MyComponent extends Component {
  someArg = 123;
}
```
```hbs
{{! app/components/my-component.hbs }}
{{this.someArg}}
```

But this method no longer works in Ember Octane for a few reasons:

1. Template-only components have no backing class to assign the value to
2. Glimmer components refer to arguments on `args`, which are read-only and
   cannot be overridden
3. Named arguments syntax bypasses JavaScript entirely in the VM, ensuring that
  `{{@someArg}}` always refers to the passed in value.

Default values have become one of the most commonly asked about questions for
users who are converting components to Octane because of this. Typically, the
suggested solutions are to either:

1. Create a getter which provides the default value if the argument is
   undefined:

   ```js
   import Component from '@glimmer/component';

   export default class MyComponent extends Component {
     get someArg() {
       return this.args.someArg ?? 123;
     }
   }
   ```

2. Use a helper to provide the default value directly where it is used in the
   template:

   ```hbs
   {{! using ember-truth-helpers }}
   {{or @someArg 123}}
   ```

The first method is limited to being used in Glimmer components, or forces users
to refactor to a Glimmer component if they are using a Template-only component.
In addition, it requires users to think regularly about whether or not they
should using `@someArg` or `this.someArg` when referencing the value, an easy
mistake to make.

The second method requires users to install an additional library or write their
own helper, as no built-in helpers exist for this purpose. In addition, they
have to either remember to use it for every single usage, or they have to use
the `{{let}}` helper to provide it:

```hbs
{{or @someArg 123}}
<MySubComponent @someArg={{or @someArg 123}} />

{{! or... }}

{{#let (or @someArg 123) as |someArg|}}
  {{someArg}}
  <MySubComponent @someArg={{someArg}} />
{{/let}}
```

In either case, the user has to write a decent amount of boilerplate, and has to
actively and regularly think about which values have defaults and which do not.
It also increases the overhead in refactoring - for instance, to refactor from a
Glimmer component to a Template-only component now requires users to convert a
number of getters to `let`s or helper usages in the template, and vice-versa.

Allowing users to provide default argument values would make this experience
much smoother overall, and can be done in a way which does _not_ invalidate the
wins we have gotten from separating arguments from internal component state in
Octane. The main issue with previously with default argument values in Classic
components was that they were _also_ mixed in with the same namespace as local
state. Consider the equivalent in JavaScript - imagine if in this function, the
local values declared at the top of the function optionally received the
argument values:

```js
function foo() {
  let a = 1;
  let b = 2;
  let localState = 3;

  return a + b + localState;
}

foo({ a: 4, b: 5 }); // 12
```

In this world, there would be no way to tell locally within the function whether
a value was entirely local, or it came from the outside. There was no separation
of internal and external state, and this is where we were with Classic
components.

Allowing argument values to provide defaults, _without_ allowing them to
override local values, is more like the way default values in JavaScript
functions actually work:

```js
function foo(a = 1, b = 2) {
  let localState = 3;

  return a + b + localState;
}

foo(4, 5); // 12
```

Here we can clearly see which values are internal and external, and also which
values are potentially defaulted and what they're defaulted to. Default values
in components would work the same way, without the ability to mix with local
component state, and with default values clearly visible for users to see.

There are a number of different possible high level APIs for defaults that we
could introduce to components. For instance, default args could be specified as
a static class field on Glimmer components:

```js
import Component from '@glimmer/component';

export default class MyComponent extends Component {
  static defaultArgs = {
    a: 1,
    b: 2,
  };
}
```

But this would not work well for Template-only components. All potential designs
will also likely be influenced by the upcoming changes to introduce Template
Imports, which _itself_ has not been entirely figured out. For instance,
default could be associated by passing them into the template definition:

```js
const defaultArgs = {
  a: 1,
  b: 2,
};

// Using the hbs`` style
export default hbs({ defaultArgs })`
  Hello, world!
`;

// Using a possible custom syntax
<template @defaultArgs={{defaultArgs}}>
  Hello, world!
</template>
```

As such, this RFC is not proposing a high level API to be added directly.
Instead, we propose adding a `getDefaultArgs` hook to component managers in
order to provide a standard place for argument defaults to be defined. This hook
is meant to be a low level primitive, which can be used by advanced users and
addon authors to explore high level APIs and ways to add default arguments.

## Detailed design

The `getDefaultArgs` hook will be added to each of the component manager
interface with the following signature:

```ts
interface DefaultArgumentValue =
  | string
  | number
  | boolean
  | null
  | undefined
  | readonly Array<unknown>
  | ReadOnly<unknown>
  | () => unknown;

interface DefaultArguments {
  named?: Record<string, DefaultArgumentValue>;
  positional?: DefaultArgumentValue[];
}

interface ComponentManager {
  getDefaultArgs(definition: object): DefaultArguments | null;
}
```

This hook receives the definition of the component as its first and only
argument, and returns an object containing default argument values. Valid
default argument values are either:

1. Primitive JavaScript values
2. Frozen arrays or object
3. Functions which return a default value

The default value will be used for the corresponding named and positional
arguments if the argument that was passed in is either `null` or `undefined`.
This matches the semantics in templates, where `null` and `undefined` generally
mean the same thing (for instance, in template interpolation). If the value is a
function, it will be called, and its return value will be used as the default
value.

So for instance, with the following `getDefaultArgs` definition:

```js
class MyManager {
  getDefaultArgs() {
    return {
      named: {
        a: 1,
        b: () => 2,
      }
    }
  }
}
```

And this template:

```hbs
{{! app/components/my-custom-component.hbs }}
<div>a: {{@a}}</div>
<div>b: {{@b}}</div>
```

And this invocation

```hbs
<MyCustomComponent @a={{3}} />
```

Would result in the following output:

```hbs
<div>a: 3</div>
<div>b: 2</div>
```

When the values passed into the component change, the new value is checked, and
if it is undefined or null then the default value is used instead. If the
default value is a function, it is called each time the argument changes to
`null` or `undefined`.

There are three goals with this design.

1. Default arguments can be known based on the definition alone. There is no
   need to consider instance state, and the two are completely separated. This
   prevents any sort of difficult to reason about mixing between the two.

2. Default arguments are a consistent, constant _shape_. That is to say, you
   cannot add or remove named arguments or positional arguments to defaults
   during subsequent updates. This is important, because the VM will emit static
   opcodes based on the arguments that are passed to a component today. If we
   cannot determine the default arguments for a component ahead of time, there
   is no way we would be able to emit static opcodes and optimizations, and we
   would instead have to run code dynamically to determine what the arguments
   are.

3. Default arguments can be determined _independently_. In particular, for
   default argument functions, which return a new value each time, we can call
   each function independently, so we are not creating unnecessary values when
   we create a default value.

### Defaults for Existing Built-ins

The goal of this RFC is to enable experimentation with argument defaults, but
providing only a custom component manager hook to do so makes it very difficult.
In order to add defaults to Template-only components or Glimmer components,
for instance, users would have to re-implement the managers for both of these
exactly just to add their custom validation hook. This would limit the
experimentation significantly, as it would require a much larger investment from
users to adopt a validation system.

In order to enable this exploration to be conducted orthogonally to the actual
component APIs, we also propose a few changes to the default components that
Ember ships with.

#### Extending `templateOnly`

Today, users can define a template only component with the
[`templateOnly`](https://github.com/emberjs/ember.js/blob/v3.23.1/packages/%40ember/component/template-only.ts#L35)
API. This API is generally meant to be a compile target which users do not
actually write, and as such it is the perfect place for us to add our
`defaultArgs` integration. Since users are not meant to write this directly, it
does not establish an opinionated high-level API.

Currently, this function optionally receives the module name of the associated
component as its first parameter. We propose extending this API so that it can
receive an options object as the first parameter with the following interface:

```ts
function templateOnly(options: {
  moduleName?: string;
  defaultArgs?: DefaultArguments
}): TemplateOnlyComponent;
```

This API will allow us to incorporate future changes and additions to the
`templateOnly` definition without needing to worry about the order becoming
unintuitive or difficult to learn. If we wish to save bytes here, the object
can be compiled down to a smaller representation in the future.

#### Extending Glimmer components

Glimmer components, as noted above, do not have a way to define default
arguments today, so we propose adding an temporary API that makes them capable
of doing so.

```ts
function setDefaultArgs(
  definition: object,
  defaultArgs: Record<string, DefaultArgumentValue>
): void;
```

Since Glimmer components can only receive named arguments, the function will
only require that users associate an object containing named argument defaults.

This function will be importable from `@glimmer/component`. It will work _only_
with Glimmer components. For custom managers provided by addons, they
should define their own APIs for associating default args. For Classic
components, users can provide default arguments on the class definition as they
could historically.

The `defaultArgs` object associated with a component class will be inherited
by all subclasses of that component, similar to how manager and template
inheritance works today.

## How we teach this

Managers are generally only described in the API documentation, and the new
additions to existing components are meant to be low-level compile targets. As
such, the only additions to documentation will be API docs.

### API Docs for component managers

The following API docs for the `validateArgs` hook should be integrated into the
API docs for component managers.

#### `getDefaultArgs`

The `getDefaultArgs` hook is an optional hook on component managers. The hook
receives the component definition as its first argument, and it can return an
object containing default argument values for the component.

```js
class MyManager {
  getDefaultArgs(definition) {
    return {
      named: {
        a: 1,
        b: () => [],
      },
      positional: ['foo']
    };
  }
}
```

The return value should either be `null` if no default arguments exist, or an
object containing `named` and/or `positional` properties with defaults for the
respective types of arguments. `named` should be a dictionary of key/default
value pairs, and `positional` should be an array of default argument values.

The following types of values can be defaults:

1. Primitive values such as string, booleans, numbers, and `null`/`undefined`.
2. Frozen arrays and objects
3. Functions which return default values

If a default value is a function, then that function will be called and its
return value used as the default value instead. This way new default values are
created per-instance.

These values will be used as default values whenever their corresponding
arguments equal `null` or `undefined`. If the value updates to become `null` or
`undefined` later on, the default value will be used then as well. If the
default value is a function, it will be called each time the value updates to
become `null` or `undefined`, producing a new default value.

### `templateOnly` additions

`templateOnly` can receive an options object as its first parameter, which can
include the following properties:

* `moduleName`: A string which indicates the module that the component is
  defined in, useful for debugging purposes.

* `defaultArgs`: An object containing default arguments for this component,
  containing `named` and/or `positional` properties with defaults for the
  respective types of arguments. `named` should be a dictionary of key/default
  value pairs, and `positional` should be an array of default argument values.

  ```js
  let MyComponent = templateOnly({
    defaultArgs: {
      named: {
        a: 1,
        b: () => [],
      },
      positional: ['foo']
    },
  });
  ```

  The following types of values can be defaults:

  1. Primitive values such as string, booleans, numbers, and `null`/`undefined`.
  2. Frozen arrays and objects
  3. Functions which return default values

  If a default value is a function, then that function will be called and its
  return value used as the default value instead. This way new default values are
  created per-instance.

  These values will be used as default values whenever their corresponding
  arguments equal `null` or `undefined`. If the value updates to become `null` or
  `undefined` later on, the default value will be used then as well. If the
  default value is a function, it will be called each time the value updates to
  become `null` or `undefined`, producing a new default value.

### `setDefaultArgs` documentation

The `setDefaultArgs` function can be used to associated default arguments with
a Glimmer component.

```js
import Component, { setDefaultArgs } from '@glimmer/component';

class MyComponent extends Component {}

setDefaultArgs(MyComponent, {
  a: 1,
  b: () => [],
});
```

The value should be a dictionary of key/default value pairs, with keys being the
argument names and values being the default value for that argument. The
following types of values can be defaults:

1. Primitive values such as string, booleans, numbers, and `null`/`undefined`.
2. Frozen arrays and objects
3. Functions which return default values

If a default value is a function, then that function will be called and its
return value used as the default value instead. This way new default values are
created per-instance.

These values will be used as default values whenever their corresponding
arguments equal `null` or `undefined`. If the value updates to become `null` or
`undefined` later on, the default value will be used then as well. If the
default value is a function, it will be called each time the value updates to
become `null` or `undefined`, producing a new default value.

## Drawbacks

- Adds additional complexity to the implementation of component managers and
  components in general.

- Using functions to generate a value argument can lead to very verbose
  statements, particularly if the user wishes to have a function be the default
  value.

  ```js
  getDefaultArgs() {
    return {
      foo() {
        // return a no-op
        return () => {};
      }
    }
  }
  ```

- Adds complexity to the mental model that has been removed recently with the
  transition to Octane. Even if the current model requires a decent amount of
  boilerplate, it is simple and predictable, and easy to reason about in
  isolation. Default values could make it somewhat more complex to do so.

## Alternatives

- We could avoid adding any ability to add default values to template-only and
  Glimmer components, and just add the manager hooks. This would make
  experimenting with high-level default arguments APIs specifically built for TO
  and Glimmer components very difficult.

- We could add a high-level API directly to Glimmer components, such as a
  `static defaultArgs` field on class definitions. This would establish
  a more conventional approach immediately, which may or may not be ideal,
  depending on where we end up with default args after initial
  experimentation.

- Default arguments could a function that returns an object instead of an object
  directly:

  ```js
  setDefaultArgs(Component, () => {
    return {
      named: {
        a: 1,
        b: () => 2,
      },
      positional: ['foo'],
    }
  })
  ```

  There are three downsides to generating all of the named arguments at once
  like this:

  1. The shape of the arguments could potentially change over time, dynamically.
     As noted in the detailed design section, this is something the VM would not
     be able to accomodate. We could however assert against this happening,
     effectively forcing the user to always return the same shape of arguments.
  2. It would require us to rerun and reconstruct each default value whenever we
     needed just one of them, which would be expensive.
  3. We would need to call and get the shape of default arguments once before the
     component was even invoked, during the compilation phase. This would mean
     that it would be called twice for the initial render of a component, once
     which would be thrown away immediately after learning the shape.

- `setDefaultArgs` could work with any component, and we could skip
  adding a hook to managers altogether. While this could work, ultimately the
  manager approach is more in-line with the design philosophy that we have been
  establishing for how managed values are defined, and in the long run all
  managers will ideally have high level APIs for adding arg defaults, so we
  will be able to deprecate the `setDefaultArgs` function altogether.
