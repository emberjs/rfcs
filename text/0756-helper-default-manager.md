---
Stage: Accepted
Start Date: 2021-05-17
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js, Learning
RFC PR: https://github.com/emberjs/rfcs/pull/756
---

# Default Helper Manager

## Summary

Anything that can be in a template has its lifecycle managed by the manager pattern.
Today, we have known managers for `@glimmer/component`, `@ember/helper`, etc.
But what happens when the VM encounters an object for which there is no manager?,
such as a plain function? This RFC proposes a default behavior for
those unknown scenarios when it comes to _helpers_.

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

A default modifier manager will be covered in a different RFC.

## Detailed design

The logic for choosing when to use a default manager is, at a high-level:

```
if (noExistingManagerFor(X)) {
  return defaultManager;
}
```
Where X is, for the purposes of this RFC, a Helper.

_A Default Manager is not something that can be chosen by the user, but is baked in to the framework
as a default so that a user doesn't have to build something to use a non-framework-specific variant
of the three constructs: Helpers, Modifiers, and Components._

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

### How a template syntax plays in to this behavior

In the current state of templates there is some ambiguity in syntax around helper/component invocation invocation.
Below is a list exploring the various syntaxes and how the code implemented in the framework for this RFC
will react to various passed value/function/etc types. All of this is current behavior and this RFC is not proposing
a syntax change. In template strict mode, there is no ambiguity to worry about.

- `{{val}}`
  - `typeof val === 'function'`: Helper, invoked
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

## How we teach this

On the [Helper Functions](https://guides.emberjs.com/v3.26.0/components/helper-functions/) page,
We'll want to insert a section early on about how helpers can be "local" to or defined on
components and controllers. Then, once there have been examples of the local/private helper,
the existing content can continue to talk about the "global helper" -- explicitly differentiating
between the local/private helper, rather then retaining the general "Helper Function" title as
"Helper Functions" are all of these, rather than just the globally defines ones.

Existing users of Ember should be made aware of these capabilities, once implemented, via
the release blog post, with some examples -- but folks watching developments in the ecosystem
will likely be aware of ember-could-get-used-to-this, which implement
some parts of this RFC, so the migration path for users of ember-could-get-used-to-this should
be straight-forward.  That addon _may_ want to deprecate their similar functionality after
the proposed behavior here lands -- though, this RFC suggests implementation such that developers
may still use ember-could-get-used-to-this without disruption.


## Drawbacks

There would no longer be a possibility of using functions as components when invoked with curlies.
Angle bracket function components would still be possible.

There could be some awkwardness around the last argument passed to the function, as the
type signature may not match how the call-site expects the signature to be.
Projects like [Glint](https://github.com/typed-ember/glint#readme) would be essential for
helping with clarity here.

The difference between `<Component @handler={{this.handler}} />` and `<Component @handler={{this.handler 1}} />`
could awkward for folks, as the syntax says that a value _always_ passes
a value, _unless_ there are arguments added within the curly braces, in which `this.handler` is invoked
and the return value is instead passed as `@handler`. This could lead to infinite revalidation assertions,
which, without an ErrorBoundary (from React) and Error messages that show the location in the template
where an error originates from, would be fairly hard to track down. The problem goes away entirely in
template strict mode, so it may not be something we want to worry about in the short-term (outside of
documenting the possibility and what to do when the situation occurs)

## Alternatives

Class-based default helpers would allow greater flexibility for creating helpers, but would also
perpetuate the current problem of most surprise when trying to invoke functions from defined outside
of ember within templates (such as XState's state.matches function).

## Unresolved questions

TBD
