---
Start Date: 2019-11-08
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/554
Tracking: (leave this empty)

---

# Deprecate getWithDefault

## Summary

Deprecate support for `getWithDefault` in Ember's Object module (@ember/object) – both the [function](https://api.emberjs.com/ember/release/functions/@ember%2Fobject/getWithDefault) and the [class method](https://api.emberjs.com/ember/release/classes/EmberObject/methods/getWithDefault?anchor=getWithDefault) – because its expected behaviour is confusing to Ember developers.

## Motivation

The problem with `getWithDefault` is that its behaviour is confusing to Ember developers. The API will only return the default value when the value of the property retrieved is `undefined`. This behaviour is often overlooked when using the function where a developer might expect that `null` or other _falsy_ values will also return the default value.

Given the JavaScript language will soon (currently in Stage 3) give us the appropriate tool for this use case using the [Nullish Coalescing Operator `??`](https://github.com/tc39/proposal-nullish-coalescing), we can deprecate usage of `getWithDefault` and use that instead.

## Transition Path

Ember will start logging deprecation messages for `getWithDefault` usage. 

We can codemod our current usage of `getWithDefault` with the equivalent behaviour using plain JavaScript. The migration guide will cover this example:

Before:

```js
import { getWithDefault } from '@ember/object';

let result = getWithDefault(obj, 'some.key', defaultValue);
```

After:

```js
import { get } from '@ember/object';

let result = get(obj, 'some.key');
if (result === undefined) {
  result = defaultValue;
}
```

#### Using Nullish Coalescing Operator

We cannot codemod directly into the nullish coalescing operator since the expected behaviour of `getWithDefault` is to only return the default value if it is strictly `undefined`. The nullish coalescing operator accepts either `null` or `undefined` to show the default value.

The function `getWithDefault` **will not return** the default value if the provided value is `null`. The function will **only return** the default value for `undefined`:

```js
let defaultValue = 1;
let obj = {
  nullValue: null,
  falseValue: false,
};

// Returns defaultValue 1, undefinedKey = 1
let undefinedValue = getWithDefault(obj, 'undefinedKey', defaultValue);

// Returns null, nullValue = null
let nullValue = getWithDefault(obj, 'nullValue', defaultValue);

// Returns obj's falseValue, falseValue = false
let falseValue = getWithDefault(obj, 'falseValue', defaultValue);
```

The nullish coalescing operator (`??`) **will return** the default value when the provided value is `undefined` or `null`:

```js
let defaultValue = 1;
let obj = {
  nullValue: null,
  falseValue: false,
};

// Returns defaultValue 1, undefinedKey = 1
let undefinedValue = get(obj, 'undefinedKey') ?? defaultValue;

// Returns defaultValue 1, nullValue = 1
let nullValue = get(obj, 'nullValue') ?? defaultValue;

// Returns obj's falseValue, falseValue = false
let falseValue = get(obj, 'falseValue') ?? defaultValue;
```

This can be an option if we are aware that either `null` or `undefined` should return the default value.

Tooling Support:

- [Babel](https://babeljs.io/) already supports the [nullish coalescing operator](https://babeljs.io/docs/en/next/babel-plugin-proposal-nullish-coalescing-operator.html) so we can use that for future use cases where we need to check if a property is `null` or `undefined` before applying a default value.

- [TypeScript](https://github.com/microsoft/TypeScript), similarly, as of [version 3.7](https://devblogs.microsoft.com/typescript/announcing-typescript-3-7/#nullish-coalescing) also supports the operator so we will not be breaking that flow either.

#### Using Object Destructuring With Defaults

If we would like to return the default value if the existing value is `undefined` we can also use [object destructuring](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment) with defaults.

Object destructuring with defaults **will return** the default value when the provided value is `undefined`:

```js
let defaultValue = 1;
let obj = {
  nullValue: null,
  falseValue: false,
};

// Returns defaultValue 1, undefinedKey = 1
let { undefinedKey = defaultValue } = obj;

// Returns defaultValue 1, nullValue = null
let { nullValue = defaultValue } = obj;

// Returns obj's falseValue, falseValue = false
let { falseValue = defaultValue } = obj;
```

## How We Teach This

Add the transition path to the [Ember Deprecation Guide](https://deprecations.emberjs.com/).

The references to `getWithDefault` will need to be removed from the [API docs](https://api.emberjs.com/ember/release/functions/@ember%2Fobject/getWithDefault). 

There are no changes needed for the [Ember Guides](https://guides.emberjs.com/release/) since we do not use it anywhere.

## Drawbacks

The downside to deprecating `getWithDefault` would be an increase to the line length of component files that use it. This change will also cause some deprecation noise but could be mitigated with a codemod.

## Alternatives

### Adding `null` as a condition

We could add `null` as a condition alongside `undefined` which would return the default value provided. This is similar to what is proposed in [Nullish Coalescing for JavaScript](https://github.com/tc39/proposal-nullish-coalescing). This would however still be a breaking change since people who are depending on `getWithDefault` to work the way it does for `null` today will be broken if we change it.

### Do nothing

We could keep support in place, and provide more guidance around using it. There are already [some](https://dockyard.com/blog/2016/03/18/get-with-default) articles cautioning usage of `getWithDefault` when dealing with `null` or _falsy_ values.

## Unresolved questions

None at the moment.
