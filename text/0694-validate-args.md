---
Stage: Accepted
Start Date: 2020-12-26
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/694
---

# Argument Validation Primitives

## Summary

Add a `validateArgs` hook to component, modifier, and helper managers which can
be used to run development time assertions against arguments.

## Motivation

It is common practice to use Ember's built in `assert` function to validate
conditions at various points today, like so:

```js
import { assert } from '@ember/debug';

function add(a, b) {
  assert('Must pass add two numbers', typeof a === 'number' && typeof b === 'number');

  return a + b;
}
```

Ember removes these assertions is production builds, so they are a zero-cost way
to make dynamic code safer in general to use, and to give users of an API an
early warning when they are misusing something.

One common way to use `assert` is to validate the arguments that are passed into
a component. In Classic components, it was possible to run assertions in a
lifecycle hook which would rerun whenever the arguments changed, such as
`didReceiveAttrs`, but in Glimmer components no such lifecycle hook exists, and
users have been forced to rely on wrapping arguments in getters in order to
validate them. In addition, Template-only components have no lifecycle hooks or
state at all, and the only way to provide assertions for them is to convert them
into a class-based component of some kind, or via helpers.

Asserting invariants about the arguments passed to a component can be a very
useful way for developers to communicate intent and ensure that their components
are being used correctly. Without a way to do this for the standard components
that users work with on a day to day basis, solutions must instead be ad-hoc
and end up fragmenting depending on specific use cases. In addition,
standardizing the community on a shared validation solution specifically for
arguments means that in time, we can begin to build tooling on top of that
solution, such as tooling that extracts API documentation for components.

There are a number of different possible high level APIs we could introduce to
components. For instance, argument validation could happen in templates via a
helper:

```hbs
{{validate-type @arg1 "string"}}
{{validate-type @arg2 "number"}}
```

Or could be associated via a static property on class based components:

```js
class MyComponent extends Component {
  static argTypes = {
    arg1: 'string',
    arg2: 'number',
  };
}
```

We could rely on existing libraries, such as the [prop-types](https://www.npmjs.com/package/prop-types)
library, or we could develop our own. The design space here is particularly
affected by the upcoming changes to introduce Template Imports, which _itself_
has not been entirely figured out. For instance, validations could be associated
by passing them into the template definition:

```js
const argTypes = {
  arg1: 'string',
  arg2: 'number',
};

// Using the hbs`` style
export default hbs({ argTypes })`
  Hello, world!
`;

// Using a possible custom syntax
<template @argTypes={{argTypes}}>
  Hello, world!
</template>
```

As such, this RFC is not proposing a high level API to be added directly.
Instead, we propose adding a `validateArgs` hook to component managers in order
to provide a standard place for argument validations to run. This hook is meant
to be a low level primitive, which can be used by advanced users and addon
authors to explore high level APIs and ways to add validations. It would also be
added to modifier managers and helper managers, which could also benefit from
the same standardized tooling that we can develop for validations as a
community.

### Overlap with TypeScript

There is a decent amount of overlap with the goals of these types of validations for
component APIs, and the goals of type systems such as TypeScript, particularly
with the `prop-types` like APIs. However, there are still cases which TypeScript
cannot solve:

1. Providing API safety/guarantees for plain JavaScript users. This is true even
   for TypeScript addon authors, as they may have plain JavaScript users.
2. Validating non-type invariants, such as args that are relative to each other
   (e.g. `@max` must be larger than `@min`)

So even for TypeScript users, argument validation can still provide benefits.
Between this, and the fact that Ember is committed to providing a first-class
user experience to JavaScript users, it still makes sense to provide this hook
independent of any potential TypeScript support.

## Terminology

For the remainder of this RFC, the term "invokable" will be used to refer to
component/modifier/helper, since all three will be validated in the same way
without any differences.

## Detailed design

The `validateArgs` hook will be added to each of the invokable manager
interfaces. It will have the following signature in each case:

```ts
interface Arguments {
  named: Record<string, unknown>;
  positional: unknown[];
}

interface InvokableManager {
  validateArgs(definition: object, args: Arguments): void;
}
```

This hook receives the definition of the invokable as its first
argument, and the args passed as the second argument. It should not return any
values, and should throw an error if the arguments are not valid. It also will
only run in development builds, and will be completely inert otherwise.

This API design accomplishes two main things:

1. It requires users to associate their validations with the _definition_ of the
   invokable, rather than the instance of it. This will prevent users from
   developing APIs that rely on instance state, and allows validations to be
   associated with invokables which do not have instance state, such as
   Template-only components. If it is determined that instance state would be
   useful in validations, then it can be added in a future RFC via a new
   capabilities version.
2. It enables general validations against the shape of arguments, rather than
   specific argument values. For instance, users can assert that extra arguments
   are not passed to a component, only the required ones. They can also assert
   relative values, such as `@max` being greater than `@min`.

This hook will be called once when the component is first invoked, and is
autotracked. It will be called again whenever the values that are validated and
tracked change, _on the next argument access_. This means that if the arguments
are updated, but no arguments are used at all within the component, then
validations _may not_ run again. This is due to the fact that argument usage is
lazy, and so we cannot know the values until they are used.

While `validateArgs` will be autotracked on its own, it will not be _consumed_
by the external tracking contexts. This prevents it from impacting the tracking
semantics of the application, and for instance causing a component rerender in
development when it would _not_ in production, which should help prevent tricky
bugs. It also means that the validations will not rerun if a tracked value that
is _only_ used by the validations changes. Since validators currently only have
access to the definition and arguments, it is unlikely that this will be an
issue since other external state, such as services, cannot be accessed.

### Validations for Existing Built-ins

The goal of this RFC is to enable experimentation with argument validations, but
providing only a custom component manager hook to do so makes it very difficult.
In order to add validations to Template-only components or Glimmer components,
for instance, users would have to re-implement the managers for both of these
exactly just to add their custom validation hook. This would limit the
experimentation significantly, as it would require a much larger investment from
users to adopt a validation system.

In order to enable this exploration to be conducted orthogonally to the actual
component APIs, we also propose a few changes to the default invokables that
Ember ships with.

#### Extending `templateOnly`

Today, users can define a template only component with the
[`templateOnly`](https://github.com/emberjs/ember.js/blob/v3.23.1/packages/%40ember/component/template-only.ts#L35)
API. This API is generally meant to be a compile target which users do not
actually write, and as such it is the perfect place for us to add our
`validateArgs` integration. Since users are not meant to write this directly, it
does not establish an opinionated high-level API.

Currently, this function optionally receives the module name of the associated
component as its first parameter. We propose extending this API so that it can
receive an options object as the first parameter with the following interface:

```ts
function templateOnly(options: {
  moduleName?: string;
  validateArgs?(definition: object, args: Arguments): void;
}): TemplateOnlyComponent;
```

This API will allow us to incorporate future changes and additions to the
`templateOnly` definition without needing to worry about the order becoming
unintuitive or difficult to learn. If we wish to save bytes here, the object
can be compiled down to a smaller representation in the future.

Since `validateArgs` is known to not run in production builds, it will be
stripped from `templateOnly` options in production builds.

#### Extending Glimmer components

Glimmer components, as noted above, do not have a way to run validations
predictably today, so we propose adding an temporary API that makes them capable
of doing so.

```ts
function setValidateArgs(
  definition: object,
  validateArgs: (definition: object, args: Arguments) => void
): void;
```

This function will be importable from `@glimmer/component`. It will work _only_
with Glimmer components. For custom managers provided by addons, they
should define their own APIs for associating args validation. For Classic
components, users can validate arguments in the existing `didReceiveAttrs` hook.

The `validateArgs` function associated with a component class will be inherited
by all subclasses of that component, similar to how manager and template
inheritance works today.

Since `validateArgs` is known to not run in production builds, all usages of
`setValidateArgs` will be stripped from production builds.

## How we teach this

Managers are generally only described in the API documentation, and the new
additions to existing components are meant to be low-level compile targets. As
such, the only additions to documentation will be API docs.

### API Docs for component/modifier/helper managers

The following API docs for the `validateArgs` hook should be integrated into the
existing API docs for the various types of managers. Substitute out the word
"component" for the manager type.

#### `validateArgs`

The `validateArgs` hook is an optional hook on component managers. It can be
used to validate the arguments that are passed to a component. If it finds that
the arguments are invalid, it should throw an error with a message describing
why they are invalid.

The hook receives the component definition as its first argument, and the
arguments themselves as its second argument.

```js
class MyManager {
  validateArgs(definition, args) {
    /**
     * Types associated as a static field, e.g.
     *
     * class MyComponent {
     *   static argTypes = {
     *     foo: 'string',
     *   };
     * }
     */
    let types = definition.argTypes;

    for (arg in types) {
      if (typeof args.named[arg] !== types[arg]) {
        throw new Error(`${arg} was incorrect type, expected ${types[arg]} but got: ${args.named[arg]}`);
      }
    }
  }
}
```

The `validateArgs` hook will run once when the component is initially created,
and is autotracked. It will rerun whenever any of the validated values change,
the next time arguments are used or accessed. It is possible validations will
not run at all if arguments are not used or accessed at all.

The `validateArgs` hook only runs in development builds, and is inert in
production builds.

### `templateOnly` additions

`templateOnly` can receive an options object as its first parameter, which can
include the following properties:

* `moduleName`: A string which indicates the module that the component is
  defined in, useful for debugging purposes.
* `validateArgs`: A validation function that validates the arguments for this
  Template-only component. The function receives the definition of the
  component, along with the arguments to the component instance. If the
  arguments are invalid, it should throw an error describing why they are
  invalid.

  ```js
  let MyComponent = templateOnly({
    validateArgs(definition, args) {
      if (typeof args.named.foo !== 'string') {
        throw new Error(`@foo was incorrect type, expected string but got: ${args.named.foo}`);
      }
    }
  })
  ```

  The `validateArgs` function will run once when the component is initially
  created, and is autotracked. It will rerun whenever any of the validated
  values change, the next time arguments are used or accessed. It is possible
  validations will not run at all if arguments are not used or accessed at all.

  This method is only run in development builds, and is inert in production
  builds.

### `setValidateArgs` documentation

The `setValidateArgs` function can be used to associated an argument
validation method with Glimmer components. The function receives the
definition of the component as its first argument, and the args passed to the
instance of the component as their second argument. If the arguments are
invalid, it should throw an error describing why they are invalid.

```js
import Component, { setValidateArgs } from '@glimmer/component';

class MyComponent extends Component {
  static argTypes = {
    foo: 'string',
  };
}

function validateArgTypes(definition, args) {
  // Access the static field on the definition.
  let types = definition.argTypes;

  for (arg in types) {
    if (typeof args.named[arg] !== types[arg]) {
      throw new Error(`${arg} was incorrect type, expected ${types[arg]} but got: ${args.named[arg]}`);
    }
  }
}

setValidateArgs(MyComponent, validateArgTypes);
```

The function will run once when the component is initially created, and is
autotracked. It will rerun whenever any of the validated values change, the next
time arguments are used or accessed. It is possible validations will not run at
all if arguments are not used or accessed at all.

The function only runs in development builds, and is inert in production builds.

## Drawbacks

- Adds additional complexity to the implementation of component, modifier, and
  helper managers.

- Having a less opinionated API means that it will take longer for the community
  to build up norms and tooling around validations, if it will even be possible.

- Allowing developers to validate against the entire args object means that they
  can write fairly complex validations. These might be more difficult to
  abstract in general.

- The fact that validating arguments is untracked by the larger context could
  mean that some validations would not rerun, even if a value used in them
  changes. This could be confusing to developers if they are relying on complex
  validations.

## Alternatives

- Validation could be limited to just components for the time being. It is most
  useful in components, especially now that TO-components and Glimmer components
  both lack the proper lifecycle hooks to add them.

- Validation could be an tracked operation overall. This would mean validations
  would affect the tracking semantics of the application, which could cause
  subtle bugs. For instance, a component might cause a rerender in development
  mode due to validations, but not in production, and this difference in
  behavior could matter (e.g. lifecycle hooks of parent components are
  triggered in development but not production).

- We could avoid adding any ability to add validations to template-only and
  Glimmer components, and just add the manager hooks. This would make
  experimenting with high validation APIs specifically built for TO and Glimmer
  components very difficult.

- We could add a high-level API directly to Glimmer components, such as a
  `static validateArgs()` function on class definitions. This would establish
  a more conventional approach immediately, which may or may not be ideal,
  depending on where we end up with validations libraries after initial
  experimentation. `validateArgs` may be ideal as is, but something like
  `prop-types` may be better.

- Instead of providing a general purpose `validateArgs` function, we could
  provide a more `prop-types` like mapping from arg to validator function. This
  would be less flexible, but more opinionated and conventional.

- `setValidateArgs` could work with any component, and we could skip
  adding a hook to managers altogether. While this could work, ultimately the
  manager approach is more in-line with the design philosophy that we have been
  establishing for how managed values are defined, and in the long run all
  managers will ideally have high level APIs for adding arg validation, so we
  will be able to deprecate the `setValidateArgs` function altogether.
