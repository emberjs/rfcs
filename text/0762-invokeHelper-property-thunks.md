---
Stage: Accepted
Start Date: 2021-08-11
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/762
---

<!---
Directions for above:

Stage: Leave as is
Start Date: Fill in with today's date, YYYY-MM-DD
Release Date: Leave as is
Release Versions: Leave as is
Relevant Team(s): Fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies
RFC PR: Fill this in with the URL for the Proposal RFC PR
-->

# Add argument-based thunks to invokeHelper

## Summary

Add conditional behavior to `invokeHelper`'s third argument such that consumers of `invokeHelper` can
have more fine-grained control over which arguments to entangle with. Today, all tracked data used
to `invokeHelper`'s third argument is entangled with the entirety of the helper.


## Motivation

`invokeHelper` (from `@ember/helper` (re-export from `@glimmer/runtime`), introduced in [RFC 626](https://github.com/emberjs/rfcs/pull/626))
is a low-level utility for helping library-authors create reactive wrapping abstractions, such as _Resources_.

However, `invokeHelper`'s third argument, the thunk, entangles all tracked data during creation
of the helper and does not allow lazy-entanglement upon access. This RFC proposes a solution
to enable lazy-entanglement of tracked data when using `invokeHelper`.

This would look like the following:
```js
import Component from '@glimmer/component';
import { cached } from '@glimmer/tracking';
import { invokeHelper } from '@ember/helper';
import { getValue } from '@glimmer/tracking/primitives/cache';

export default class MyComponent extends Component {
  @tracked left = 1;
  @tracked right = 2;

  // currently, all arguments are conusmed at once (this RFC is not suggesting removing the current behavior)
  oldAdder = invokeHelper(this, Adder, () => ({ 
    named: { operation: this.op },
    positional: [this.left, this.right],
  }));

  // arguments are individually entangleable upon access
  adder = invokeHelper(this, Adder, {
    named: {
      operation: () => this.op;
    },
    positional: [
      () => this.left,
      () => this.right,
    ],
  });

  get adderValue() {
    return getValue(this.adder);
  }

  get oldAdderValue() {
    return getValue(this.oldAdder);
  }
}

// Assume there is a Helper manager registered that knows what to do with this
// and getValue(this.calculator) returns an instance of this class
// (instead of calling compute like on the default Helper class)
class Adder {
  @cached
  get leftDoubled() {
    console.log('doubling the left');
    return this.args.positional[0] * 2;
  }

  @cached
  get rightDoubled() {
    console.log('doubling the right');
    return this.args.positional[1] * 2;
  }

  get result() {
    return this.leftDoubled + this.rightDoubled;
  }
}
```
```hbs
{{this.adderValue.result}} => prints 6, both console.logs print
{{this.oldAddeerValue.result}} => prints 6, both console.logs print
{{!-- sometime later this.left changes to 3 --}}
{{this.adderValue.result}} => would print 10, only one console.log prints (because the left changed and not the right)
{{this.oldAdderValue.result}} => would print 10, both console.logs print, even though only this.left changed
```

prior to this RFC's implementation, when _any_ arg changes, everything is invalidated and usage of the
`@cached` decorator is moot, whereas intuition states that only changes to arguments accessed within
the getter are consumed.

One of the benefits of tracking, and by proxy, auto-tracking, is that data is lazily entangled.
_You only pay for what you use or consume_. However, this is not the case with `invokeHelper`.
When using `invokeHelper`, all tracked data accessed/consumed in the third arg, the thunk, is
entangled with the entirety of the helper (passed to `invokeHelper`'s second arg). In order to
provide the same lazy-entanglement benefits we have in the rest of the framework, `invokeHelper`,
needs to also support lazy-entanglement per-argument (both named and positional).

## Detailed design

This is a non-breaking, additive change to `invokeHelper`'s third argument, the thunk.
Currently, `invokeHelper`'s thunk is a single thunk:
```js
invokeHelper(context, helper, () => {
  return {
    positional: [valA, valB],
    named: {
      namedArg: valC,
    },
  };
});
```
This RFC Proposes the following alternative API:
```js
invokeHelper(context, helper, {
  positional: [
    () => valA,
    () => valB,
  ],
  named: {
    namedArg: () => valC,
  }
});
```

This change would occur in [`@glimmer/runtime/lib/helpers/invoke.ts`](https://github.com/glimmerjs/glimmer-vm/blob/master/packages/%40glimmer/runtime/lib/helpers/invoke.ts#L48)
where the the `computeArgs` type would change:
```diff
 export function invokeHelper(
   context: object,
   definition: object,
-  computeArgs?: (context: object) => Partial<Arguments>
+  computeArgs?:
+    | (context: object) => Partial<Arguments>
+    | Partial<{
+        positional: Array<(context: object) => unknown>
+        named: Record<string, (context: object) => unknown>
+      }>
 ): Cache<unknown> {
```

later on in the `invokeHelper` function,
```diff
-  let args = new SimpleArgsProxy(context, computeArgs);
+  let args;
+  if (typeof computeArgs === 'function') {
+    args = new SimpleArgsProxy(content, computeArgs);
+  } else {
+    // details tbd
+    args = new DetailedArgThunksProxy(content, computeArgs);
+  }
```
The (name tbd) `DetailedArgThunksProxy` would lazily evaluate each of the argument thunks on
access of the specific argument accessed.


## How we teach this

_Update [docs](https://api.emberjs.com/ember/3.27/functions/@ember%2Fhelper/invokeHelper) with examples in the doc-comment block in `@ember/helper/index.ts`_

Currently, this part:
> `computeArgs`: An optional function that produces the arguments to the helper. The function receives the parant context as an argument, and must return an object with a `positional` property that is an array and/or a `named` property that is an object.

Should be changed to:

`computeArgs`: Optionally, either a function that produces the arguments to the helper or an object with keys `named` and `positional` whos values are functions that produce the values for the helper.

When passing a function that produces the arguments to the helper, it should return an object that has a `named` property that has a value that is an object, and a `positional` property that has a value that is an array. This allows for a concise API, but means that all tracked data consumed in this function will be entangled with the helper, regardless of which arguments are used or when they are accessed.

Example:
```js
// ... imports and Helper ommitted for brevity
export default class PlusOne extends Component {
  plusOne = invokeHelper(this, RemoteData, () => {
    return {
      positional: [this.args.number],
    }
  });
}
```

When passing an object with `named` and `positional` arguments, each value must be a function. This function will receive the parent context as an argument and must return a single value. This allows for fine-grained control of tracked entanglement within the helper, `RemoteData` such that entanglement does not occur until the positional argument is accessed.

Example:
```js
// ... imports and Helper ommitted for brevity
export default class PlusOne extends Component {
  plusOne = invokeHelper(this, RemoteData, {
    positional: [() => this.args.number],
  });
}
```

## Drawbacks

- It looks verbose / awkward

## Alternatives

TBD?

## Unresolved questions

TBD?
