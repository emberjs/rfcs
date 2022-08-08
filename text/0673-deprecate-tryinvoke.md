---
stage: accepted
start-date: 2020-10-02T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/673
project-link:
---

# Deprecate `tryInvoke`

## Summary

Deprecate support for `tryInvoke` in Ember's Utils module (@ember/utils) because native JavaScript has [optional chaining](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining) for developers to use as an alternative solution. Deprecating `tryInvoke` will help to reduce Ember API redundancy.

## Motivation

In most cases, Function arguments should not be optional, but in the rare occasion that a Function argument is intentionally optional by design, we can use native JavaScript's [optional chaining](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining) as a solution. Deprecating `tryInvoke` will help to reduce Ember API redundancy.

## Transition Path

Ember will start logging deprecation messages for `tryInvoke` usage. Deprecation text: `Using tryInvoke has been deprecated. Instead, consider using native JavaScript optional chaining.`

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

The optional chaining operator `?.` permits reading the value of a property located deep within a chain of connected objects without having to expressly validate that each reference in the chain is valid. The `?.` operator functions similarly to the `.` chaining operator, except that instead of causing an error if a reference is nullish (`null` or `undefined`), the expression short-circuits with a return value of `undefined`. When used with Function calls, it returns `undefined` if the given function does not exist:

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

- [ember-cli-babel](https://www.npmjs.com/package/ember-cli-babel) all recent versions support optional chaining operator.

- [TypeScript](https://github.com/microsoft/TypeScript), similarly, as of [version 3.7](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#optional-chaining) also supports the operator so we will not be breaking that flow either.

## How We Teach This

### Add to Ember Deprecation Guide

In the [Ember Deprecation Guide](https://deprecations.emberjs.com/) we will add the following text:

Deprecate support for `tryInvoke` in Ember's Utils module (@ember/utils). In most cases, Function arguments should not be optional, but in the rare occasion that a Function argument is intentionally optional by design, we can use native JavaScript's [optional chaining](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining) as a solution. Deprecating `tryInvoke` will help to reduce Ember API redundancy.

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

### Remove from API docs

The references to `tryInvoke` will need to be removed from the [API docs](https://api.emberjs.com/ember/release/functions/@ember%2Futils/tryInvoke).

### Add to Ember Guides

In [Ember Guides](https://guides.emberjs.com/release/) under the [Arguments](https://guides.emberjs.com/release/components/component-arguments-and-html-attributes/) section, we will create 2 new sub-headings called `Function Arguments` and `Optional Function Arguments`:

#### Function Arguments
Arguments passed into components can be of type [Function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Functions). In most cases, Function arguments should be treated as required arguments and therefore should be invoked with normal Function invocation `()`. It is important to intentionally treat Function arguments as required because in the off chance that the Function argument is `undefined`, normal Function invocation `()` will cause a runtime exception and produce a stack trace, making it easier for the developer to find the root cause of the bug.

```js
// app/components/parent.js
@action
fooParent() {
 // ...
}
```

```hbs
{{!-- app/components/parent.hbs --}}
<Child @bar={{this.fooParent}} />
```

```js
// app/components/child.js
fooChild() {
 this.args.bar('baz');
}
```

#### Optional Function Arguments
In the rare occasion that a Function argument is intentionally optional by design, you can use native JavaScript's [optional chaining](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining) to invoke the optional Function argument `?.()`. We want to avoid unintentionally treating Function arguments as optional because optional chaining invocation has the side effect of failing silently with no stack trace logged. This will cause a difficult debugging experience for the developer.

```hbs
{{!-- app/components/parent.hbs --}}
<Child @bar={{this.fooParent}} />
```

```hbs
{{!-- app/components/some-other-parent.hbs --}}
<Child />
```

```js
// app/components/child.js
fooChild() {
 this.args.bar?.('baz');
}
```

## Drawbacks

This change will cause some deprecation noise but could be mitigated with a codemod.

## Alternative Solutions

We could check that the Function name exists on the object before invocation using an `if` block, but this alternative leaves the developer to have to wrap each Function call in an `if` block, making this pattern very cumbersome.

```js
foo() {
  if (typeof this.args.bar === 'function') {
    this.args.bar('baz');
  }
}
```

## Alternatives

#### Do nothing
We could keep support in place, and provide more guidance around using it.

## Unresolved questions

None at the moment.
