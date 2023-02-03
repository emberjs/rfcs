---
stage: recommended
start-date: 2021-05-17T00:00:00.000Z
release-date: 2022-05-13T00:00:00.000Z
release-versions:
  ember-source: v4.5.0
teams:
  - framework
  - learning
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/756'
  recommended: 'https://github.com/emberjs/rfcs/pull/866'
project-link:
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
ember-could-get-used-to-this implementing a Helper Manager that knew what to do with plain functions.

This has the impact of greatly reducing mental overhead around helpers for app and addon authors, in that,
folks no longer need to jump over to the app/addon helpers directory to create a helper
"just to do this one simple thing". It's all now `{{this.myHelper}}` or `{{this.myModifier}}`.

The introduction of a plain-function helper-manager is important because over the past several years,
we've seen on numerous occasion, folks new to Ember inherently expect that plain functions work in templates.

Example:

```js
import Component from '@glimmer/component';
import { setComponentTemplate } from '@ember/component';
import { hbs } from 'ember-cli-htmlbars';

export default class Example extends Component {
  double = num => num * 2;
}
```
```hbs
{{this.double 2}} => prints 4
<SomeComponent @foo={{this.double 2}} /> => @foo === 4
```


A default modifier manager will be covered in a different RFC.

## Detailed design

_A Default Manager is not something that can be chosen by the user, but is baked in to the framework
as a default so that a user doesn't have to build something to use a non-framework-specific variant
of the three constructs: Helpers, Modifiers, and Components._

The desired usage of a plain function in a template should be:
 - convenient
 - reduce boilerplate
 - be easily portable to JS for developers' mental model of how template and JS interact.

Which results in:
 - default to positional parameters
 - all named arguments are grouped into an "options object" as the last parameter.
   this happens to align with the _syntax_ of helper invocation where named arguments may not appear
   before the last positional argument.

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

#### Example Default Helper Implementation

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
    } else {
      argsForFn.push({});
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
 - when the "helper" is created, the function is not invoked
 - when `getValue` is invoked,
   - the function is invoked with the named arguments all grouped into the last arg
   - if no named arguments are given, an empty object is used instead to allow less nullish checking in userland
 - to register this helper manager, it should occur during app boot so developers do not need to import anything to
   trigger the `setHelperManager` call

### Updating highlevel manager choosing algorithm

This is the existing manager chooser algorithm, but with extra additions required by this RFC (notated by `-->`).

- if inside element space
    - use `getModifierManager`
- if inside document body space using curly invocation
    1. attempt lookup via `getComponentManager` and invoke it
    2. attempt lookup via `getHelperManager` and invoke it
    3. `-->` if function, fallback to this RFC's default manager
    4. render, e.g.: `[Object object]`, for objects
- if inside document body space using angle invocation
    - attempt to lookup via `getComponentManager` and invoke it
- if inside of a subexpression's "head" (e.g. `PathExpression`) position
    1. attempt to lookup via `getHelperManager` and invoke it
    2. `-->` if function, fallback to this RFC's default manager
    3. error
- if inside of a subexpression arguments
    - pass the value

### How a template syntax plays in to this behavior

In the current state of templates there is some ambiguity in syntax around helper/component invocation invocation.
Below is a list exploring the various syntaxes and how the code implemented in the framework for this RFC
will react to various passed value/function/etc types. All of this is current behavior and this RFC is not proposing
a syntax change. In template strict mode, there is no ambiguity to worry about.

- `{{val}}`
  - `typeof val === 'function'`: Helper, invoked
    - presently, it's possible to have curly components use this syntax as well. By defining this as a helper,
      there is a possibility of confusion as the manager choosing algorithm will select a component before it selects a helper
      when using curlies.
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
