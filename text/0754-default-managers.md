---
Stage: Accepted
Start Date: 2021-05-17
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js, Learning
RFC PR: https://github.com/emberjs/rfcs/pull/754
---

# Default Managers

## Summary

Anything that can be in a template has its lifecycle managed by the manager pattern.
Today, we have known managers for `@glimmer/component`, `@ember/helper`, etc.
But what happens when the VM encounters an object for which there is no manager?,
such as a plain function? This RFC explores and proposes a default behavior for
those unknown scenarios.

## Motivation

The addon, [ember-could-get-used-to-this](https://github.com/pzuraq/ember-could-get-used-to-this)
demonstrated that it's possible to use plain functions for helpers and modifiers.
And since Ember 3.25, helpers can be invoked directly from value references, this opened
a whole new world of ergonomics improvements where a dev could define a function in a
component class and use that function **as** a helper, thanks to
ember-could-get-used-to-this implementing a Helper Manager that _knew what to do with plain functions.

This has the impact of greatly reducing helper burden for apps and addon devs, in that,
folks no longer need to jump over to the app/addon helpers directory to create a helper
"just to do this one simple thing". It's all now `{{this.myHelper}}` or `{{this.myModifier}}`.

This has the added benefit of, with template strict mode, using plain functions in module-scope
for helpers and modifiers.

For Example:

<!--
  If you're reading this RFC non-rendered, ignore the "jsx" language tag.
  "jsx" is incorrect, and isn't perfect, but "gjs" syntax highlighting
  hasn't been added to GitHub's highlighting yet.
-->

```jsx
const double = num => num * 2;
const resizable = element => { /* ... */ };

<template>
  {{double 2}}
  <div {{resizable}}></div>
</template>
```
_Note: `<template>` is experimental syntax, and should not be be the focus of this example_

Another aspect of which is exciting is that it becomes easier, in tests to grab
the output of yielding components:

```jsx
test('...', async function (assert) {
  let output = [];
  const capture = (...args) => output = args;

  await render(hbs`
    <MyComponent as |yielded info and things|>
       {{capture yielded info and things}}
    </MyComponent>
  `);

  assert.equal(output[0], /* value of yielded */ );
  assert.equal(output[1], /* value of info */ );
  // ...
})
```

## Detailed design

Each implemented meta-manager needs a default manager specified. The logic for choosing when to use
a default manager is, at a high-level:

```
if (noExistingManagerFor(X)) {
  return defaultManager;
}
```
Where X is one of a Helper, Modifier, or Component

_A Default Manager is not something that can be chosen by the user, but is baked in to the framework
as a default so that a user doesn't have to build something to use a non-framework-specific variant
of the three constructs: Helpers, Modifiers, and Components._

### Helpers

Borrowing most of the implementation from ember-could-get-used-to-this with one change: the signature
of the function may receive an options-like object as the last parameter if named arguments are specified.
This aligns with the _syntax_ of helper invocation where named arguments may not appear before the last
positional argument.

#### Example with mixed params

```hbs
{{this.calculate 1 2 op="add"}}
```
would be an example invocation of a function with the following signature
expressed as TypeScript for clarity:
```ts
type Options = { op: 'add' | 'subtract' }
class A {
  calculate(first: number, second: number, options: Options) {
    // ...
  }
}
```
for unknown amounts of parameters, the [typescript can be awkward](https://www.typescriptlang.org/play?#code/JYOwLgpgTgZghgYwgAgMJwDYIK4bpAeQAcxgB7EAZ2QG8AoZR5MogLmQHI4ATbj5AD6dK2AEYcA3HQC+dMAE8iKdFlz4IAQSgBzagF5kAbQB0pkNgC2o6IYC6AGjSYceQiXJVbUujGwgEpBTICM5qkAAUpsZwOpTsKi7qWroAlLQMyBgQYMzuFPrIMbrGRCzhaXDUCWEQxIFUUoxZOeYWBUXUlcit1lB23owI+WRZxhhk2uE0ufWUjq3U0ilSsj5+AR7Boa4QAEyRph3x20mxafRN2UYss45RUBAAbtCUENy2yAYdxg-PUK-lRqZK4LT7IX4vN4-J6QwF0DJDKgjCBjCZTGYeObdSyLZYyeHwoA),

but there is a [TC39 proposal: proposal-deiter](https://github.com/tc39/proposal-deiter) that could make
destructuring simpler and inlined to
```ts
  calculate(...numbers: number[], options: Options) {
```

#### Example with only positional parameters

```hbs
{{this.add 1 2 3 4}}
```
Because there are no named arguments passed in, the method signature can be simple:
```js
class A {
  add(...numbers: number[]) {
    // ...
  }
}
```

The implementation for the this function-handling helper-manager could look like this:
```ts
import {
  setHelperManager,
  capabilities as helperCapabilities,
} from '@ember/helper';
import { assert } from '@ember/debug';

class FunctionHelperManager {
  capabilities = helperCapabilities('3.23', {
    hasValue: true,
  });

  createHelper(fn, args) {
    return { fn, args };
  }

  getValue({ fn, args }) {
    let argsForFn = args.positional;

    if (Object.keys(args.named).length > 0) {
      argsForFn.push(args.named);
    }

    return fn(...argsForFn);
  }

  getDebugName(fn) {
    return fn.name || '(anonymous function)';
  }
}

const DEFAULT_HELPER_MANAGER = new FunctionHelperManager();

// side-effect -- this file needs to be imported for the helper manager to be installed
setHelperManager(() => DEFAULT_HELPER_MANAGER, Function.prototype);
```

### Modifiers

Modifiers' default should be similar to the Helpers' default, in that they would be function-based.
The primary difference is that the first argument to a function-modifier is the attached `Element`,
and the function-modifier _may_ return a destruction function.

Example:
```js
class MyComponent {
  wiggle(element, first, second, options) {
    const doAnimation = { /* ... */ }

    element.addEventListener('touchstart', doAnimation);

    return () => element.removeEventListener('touchstart', doAnimation);
  }
}
```
```hbs
<button {{this.wiggle "first arg" "second arg"}}>
```
Similar to the helper manager, because the invocation of `{{this.wiggle ...` has no named arguments,
the last parameter in the `wiggle` function signature will be `undefined`.

The implementation of this modifier manager could look like:

```js
import {
  setModifierManager,
  capabilities as modifierCapabilities,
} from '@ember/modifier';
import { destroy, registerDestructor } from '@ember/destroyable';
import { setOwner } from '@ember/application';

class DefaultModifierManager {
  capabilities = modifierCapabilities('3.22');

  createModifier(fn, args) {
    return { fn, args, element: undefined, destructor: undefined };
  }

  installModifier(state, element) {
    state.element = element;
    this.setupModifier(state);
  }

  updateModifier(state) {
    this.destroyModifier(state);
    this.setupModifier(state);
  }

  setupModifier(state) {
    let { fn, args, element } = state;

    let argsForFn = args.positional;

    if (Object.keys(args.named).length > 0) {
      argsForFn.push(args.named);
    }

    state.destructor = fn(element, ...argsForFn);
  }

  destroyModifier(state) {
    if (typeof state.destructor === 'function') {
      state.destructor();
    }
  }
}

const DEFAULT_MODIFIER_MANAGER = new DefaultModifierManager();

// side-effect -- this file needs to be imported for the modifier manager to be installed
setModifierManager(() => DEFAULT_MODIFIER_MANAGER, Function.prototype);
```


### Components

The default component manager could be similar to the component manager for `@glimmer/component`,
but be compatible with vanilla classes, such as:
```js
class MyComponent {
  @tracked count = 0;
}
```
Since the introduction of `@ember/destroyable`, we no longer _need_ a specific class for helping
with destruction.

If a component wanted to have a destruction hook, it could register one itself during construction:
```js
import { registerDestructor } from '@ember/destroyable';

export default MyComponent {
  constructor(...args) {
    super(...args);

    registerDestructor(this, () => this.myDestroy)
  }

  myDestroy() { /* ... */ }
}
```

and the component manager to support this could look like:

```js
import { setComponentManager } from '@ember/component';
import { destroy } from '@ember/destroyable';

class DefaultComponentManager {
  constructor(owner) { this.owner = owner; }

  createComponent(ComponentClass, args) {
    return new ComponentClass(this.owner, args);
  }

  getContext(component) {
    return component;
  }

  destroyComponent(component) {
    destroy(component);
  }
}

// side-effect -- this file needs to be imported for the component manager to be installed
setComponentManager((owner) => new DefaultComponentManager(owner), Object.constructor);
```

### How a template syntax plays in to this behavior

In the current state of templates there is some ambiguity in syntax around helper/component invocation invocation.
Below is a list exploring the various syntaxes and how the code implemented in the framework for this RFC
will react to various passed value/function/etc types. All of this is current behavior and this RFC is not proposing
a syntax change. In template strict mode, there is no ambiguity to worry about.

- `{{val}}`
  - `typeof val === 'function'`: Helper
    - presently, it's possible to have curly components use this syntax as well. By defining this as a helper,
      the possibility of using functions as components using curly syntax is eliminated. Curly components could still
      be used with a base class.
    - because angle-bracket invocation is the generally accepted way to invoke a component,
      this may be an acceptable trade-off
  - `typeof val === 'object'`: Value, rendered
  - `val instanceof AnyClass`: Value, rendered
    - Today, classes are `.toString()`'d, but it's feasible that one could define their own
      custom helper manager or component manager to do something different.
- `{{ (val) }}`
  - `typeof val === 'function'`: Helper, existing behavior
  - `typeof val === 'object'`: Expected Helper error, no manager found
  - `val instanceof AnyClass`: Expected Helper, no manager found
- `{{val arg}}`
  - `typeof val === 'function'`: Helper, existing behavior
    - Similar to `{{val}}`, it is possible today to define a component manager that takes positional args that
      would be invoked with this syntax.
  - `typeof val === 'object'`: Expected Helper error
  - `val instanceof AnyClass`: Expected Helper, no manager found
- `<val />`
  - `typeof val === 'function'`: Component
    - both functions and classes have `typeof val === 'function'`, so having a default
      Component Manager that uses classes doesn't prevent the possibility of someone
      wanting to use functions as components
  - `typeof val === 'object'`: Expected Component Manager missing error
  - `val instanceof AnyClass`: Component, because any class can be a component with the proposed
    default component manager
- `<Component @arg={{val arg}} />`
  - `typeof val === 'function'`: Helper, existing behavior
  - `typeof val === 'object'`: Expected Helper error
  - `val instanceof AnyClass`: Expected Helper, no manager found
- `<Component @arg={{val}} />`
  - `typeof val === 'function'`: Value
    - if this were to evaluate as a helper it would break existing behavior where you may be
      passing an event handler to the component
    - another consequence of evaluating `val` as a helper and invoking it is that it would be
      more likely to cause an infinite revalidation assertion, which at present, would be hard
      to track down the source of, but may be an option if the VM one day has an equivalent to
      React's ErrorBoundary
    - if someone wanted to invoke `val` as a helper when passed as an argument, they would need to
      add surrounding `()`, example: `<Component @arg={{ (val) }}` />
  - `typeof val === 'object'`: Passed as argument to the component
  - `val instanceof AnyClass`: Passed as argument to the component
- `<Component @arg={{val 1}} />`
  - `typeof val === 'function'`: Helper, existing behavior
    - another consequence of evaluating `val` as a helper and invoking it is that it would be
      more likely to cause an infinite revalidation assertion, which at present, would be hard
      to track down the source of, but may be an option if the VM one day has an equivalent to
      React's ErrorBoundary
  - `typeof val === 'object'`: Expected Helper error
  - `val instanceof AnyClass`: Expected Helper, no manager found
- `<Component @arg={{ (val 1) }} />`
  - `typeof val === 'function'`: Helper, existing behavior
  - `typeof val === 'object'`: Expected Helper error
  - `val instanceof AnyClass`: Expected Helper, no manager found
- `<div {{val}}>`
  - `typeof val === 'function'`: Modifier, existing behavior
  - `val instanceof AnyClass`: Expected Modifier, no manager found
  - `typeof val !== 'function'`: Expected Modifier error
  - the behavior of modifiers is unchanged, as there is no ambiguity possible at this time

## How we teach this

### Changes to Existing Pages

On the [Helper Functions](https://guides.emberjs.com/v3.26.0/components/helper-functions/) page,
We'll want to insert a section early on about how helpers can be "local" to or defined on
components and controllers. Then, once there have been examples of the local/private helper,
the existing content can continue to talk about the "global helper" -- explicitly differentiating
between the local/private helper, rather then retaining the general "Helper Function" title as
"Helper Functions" are all of these, rather than just the globally defines ones.

### Additional Pages

This RFC adopts a default modifier manager, which means that folks can use modifiers without
the assistance of a third-party addon, such as ember-modifier. This means that we'll benefit from
adding a page to the guides on modifiers, their philosophy, when to use them, etc. While the
overall API for modifiers in general has been "not perfect", there has been a fair amount of
experimentation with modifiers since the 3.11 - 3.16 (pre-octane) era. Function modifiers are
significantly simpler than class-based modifiers, and the complexity of the specifics of class-
based modifiers is where most of the hesitance around adding modifiers by default has been.

The default component manager may not need its own guides-documentation, because the prevelance
of `@glimmer/component` is so widespread, documenting yet-another-way-to-define-a-component may
cause more harm than good.

Existing users of Ember should be made aware of these capabilities, once implemented, via
the release blog post, with some examples -- but folks watching developments in the ecosystem
will likely be aware of ember-could-get-used-to-this and ember-modifier, which both implement
some parts of this RFC, so the migration path for users of those addons should be straight-forward.
Those addons _may_ want to deprecate their similar functionality after the proposed behavior here
lands -- though, this RFC suggests implementation such that developers may still use both
addons without disruption.



## Drawbacks


For helpers and modifiers, there could be some awkwardness around the last argument
passed to the function, as the type signature may not match how the call-site expects
the signature to be. Projects like [Glint](https://github.com/typed-ember/glint#readme)
would be essential for helping with clarity here.

For components, people may begin to ask if they even need `@glimmer/component`, and that's
something that the core team may want to address -- some folks may feel like there is churn
in the component base-class, and don't know "what is correct", etc. The two approaches are
similar enough where we could introduce a migration via lint rule or codemod.

This proposal eliminates the possibility of using functions as components with the curly
bracket syntax. However, using functions as components would still be possible with angle bracket
invocation.

The difference between `<Component @handler={{this.handler}} />` and `<Component @handler={{this.handler 1}} />`
could awkward for folks, as this proposal suggests that curly braces on a component _always_ pass
a value, _unless_ there are arguments added within the curly braces, in which `this.handler` is invoked
and the return value is instead passed as `@handler`. This could lead to infinite revalidation assertions,
which, without an ErrorBoundary (from React) and Error messages that show the location in the template
where an error originates from, would be fairly hard to track down. The problem goes away entirely in
template strict mode, so it may not be something we want to worry about in the short-term (outside of
documenting the possibility and what to do when the situation occurs)

## Alternatives

There are other defaults we could have:
 - class based helpers & modifiers
 - function based-components (though this would force the introduction of new paradigms, e.g.: React's hooks)
    - React's hooks + ember's reactivity _may_ be something with exploring further and there have been a couple
      of experiments already:
      - https://github.com/rwjblue/ember-functional-component
      - https://github.com/lifeart/hooked-components
      - https://github.com/NullVoxPopuli/ember-function-component

I don't think any of the above alternatives appeal to the goal of "least surprise" when authoring each of these
constructs.

## Unresolved questions

What do we do about the `@arg={{this.function 1}}` situation? do we just let it invoke (to be consistent),
and make sure that the infinite revalidation assertion says exactly where the error originates from?


