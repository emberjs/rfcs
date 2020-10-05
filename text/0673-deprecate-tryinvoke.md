- Start Date: 2020-10-02
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/673
- Tracking: (leave this empty)

# <RFC title>

## Summary

Deprecate support for `tryInvoke` in Ember's Utils module (@ember/utils) because the Javascript language has [optional chaining](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining) for us to use as a native solution.

## Motivation

In most cases, function arguments should not be optional, but in the rare occasion that it is optional, the Javascript language has [optional chaining](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining) so we can deprecate the usage of `tryInvoke`.

## Transition Path

Ember will start logging deprecation messages for `tryInvoke` usage. 

We can codemod our current usage of `tryInvoke` with the equivalent behaviour using plain JavaScript. The migration guide will cover this example:

Before:

```js
import { tryInvoke } from '@ember/utils';
 
foo() {
 tryInvoke(this.args, 'bar', ['baz']);
}
```

After:

```js
foo() {
 this.args.bar?.('baz');
}
```

#### Using Optional Chaining Operator

The optional chaining operator `?.` permits reading the value of a property located deep within a chain of connected objects without having to expressly validate that each reference in the chain is valid. The `?.` operator functions similarly to the `.` chaining operator, except that instead of causing an error if a reference is nullish (`null` or `undefined`), the expression short-circuits with a return value of `undefined`. When used with function calls, it returns `undefined` if the given function does not exist:

```js
const adventurer = {
  name: 'Alice',
  cat: {
    name: 'Dinah'
  }
};

const dogName = adventurer.dog?.name;
console.log(dogName);
// expected output: undefined

console.log(adventurer.someNonExistentMethod?.());
// expected output: undefined
```

Tooling Support:

- [Babel](https://babeljs.io/) already supports the [optional chaining operator](https://babeljs.io/docs/en/babel-plugin-proposal-optional-chaining) so we can use that for future use-cases.

- [TypeScript](https://github.com/microsoft/TypeScript), similarly, as of [version 3.7](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#optional-chaining) also supports the operator so we will not be breaking that flow either.

## How We Teach This

Add the transition path to the [Ember Deprecation Guide](https://deprecations.emberjs.com/).

The references to `tryInvoke` will need to be removed from the [API docs](https://api.emberjs.com/ember/release/functions/@ember%2Futils/tryInvoke). 

There are no changes needed for the [Ember Guides](https://guides.emberjs.com/release/) since we do not use it anywhere.

## Drawbacks

This change will cause some deprecation noise but could be mitigated with a codemod.

## Alternatives

We could check that the function name exists on the object before invocation using an `if` block, but this alternative leaves the developer to have to wrap each function call in an `if` block, making this pattern very cumbersome.

```js
foo() {
  if (typeof this.args.bar === 'function') {
    this.args.bar('baz');
  }
}
```
## Unresolved questions

None at the moment.
