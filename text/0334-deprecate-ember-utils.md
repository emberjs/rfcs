---
stage: ready-for-release
start-date: 2018-05-22T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/334'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/1056'
project-link:
---

# Deprecate Ember Utils & dependent Computed Properties

## Summary

This RFC proposes the deprecation of the following [methods](https://api.emberjs.com/ember/4.4/modules/@ember%2Futils):
- `compare`
- `isBlank`
- `isEmpty`
- `isEqual`
- `isNone`
- `isPresent`
- `typeOf`

As well as [computed properties](http://api.emberjs.com/ember/4.4/modules/@ember%2Fobject#functions-computed) that rely on these methods:
- `notEmpty`
- `empty`
- `none`

Once deprecated, these methods will move to an external addon to allow users to still take advantage of the succinctness these utility methods provide.

## Motivation

To further Ember's goal of a svelte core and moving functionality to separate packages, this RFC attempts to deprecate a few utility methods and extract them to their own addon.  A part from the aforementioned goal, in many cases, these utility methods do not add a whole lot of value over writing plain old JavaScript.  Moreover, these methods generally have a negative cognitive impact on the code in question.  In order to reduce code liability and advocate for users to write plain JavaScript, these methods could be removed from Ember core and extracted to an addon.

A simple example of this cognitive load would be checking for the presence of a variable of type String that can only have four states - `undefined`, `null`, `''` or `'MYSTRING'`.  One might check if the variable is truthy with `!!myVariable`.  However, a utility method such as `isPresent(myVariable)` can increase the cognitive load of the truthy check in question as it checks for many different types.

In addition, when checking whether a primitive value is truthy, a user can choose from `isPresent`, `isBlank`, `isNone`, or `isEmpty`.  This can can lead to some confusion for users as there are many options to do the same thing.

Lastly, `isEmpty` documented [here](https://www.emberjs.com/api/ember/release/functions/@ember%2Futils/isEmpty) will return false for an empty object.  Defining what an empty object is can be a bit tricky and `isEmpty` provided specific semantics that may not fit a users definition.  This problem of specific semantics not fitting a users definition also applies to the other utility methods as well.

## Detailed design

`compare`, `isBlank`, `isEmpty`, `isEqual`, `isNone`, `isPresent`, `typeOf`, `notEmpty`, `empty`, and `none` will be deprecated at first with an addon that extracts these methods for users that still find value in these utility methods.  Any instantiation of one of these functions imported from `@ember/utils` will log a deprecation warning, notifying the user that this method is deprecated and provide a link to the supported addon.

Addons that previously relied on these utility methods would also be expected to install this package. Moreover, to ease adoption of the external addon, a codemod will become available for the community.

## How we teach this

ES5/ES6 brings improved language semantics. Moreover, using Ember in IE9, IE10, and PhantomJS is not supported as of Ember 3.0.  As a result, we as a community can guarantee results across browsers when using just JavaScript.  Some of the benefits from these utility methods eminated from a need to guarantee results with ES3 browsers.  Today, we can nudge users to use JavaScript where needed and install an addon in cases where they feel it is necessary.

This would require an update to the guides to indicate that the use of any of these utility methods are now available in an addon.  Moreover, in the README of the addon, we can provide specific examples of when using JavaScript is easier than reaching for one of these utility methods.

```js
// Current state
import { isEmpty } from '@ember/utils';

function isTomster(attributes) {
  if (isEmpty(attributes)) {
    return false;
  }
}
```

If you prefer to install the addon directly, we will provide a codemod to update the import path.

```js
// With an addon
import { isEmpty } from 'ember-utils';

function isTomster(attributes: any): boolean {
  if (isEmpty(attributes)) {
    return false;
  }
}
```

However, depending on the source data, this logic can be migrated to the following. Let us assume the shape of attributes is an object:

```js
function isTomster(attributes?: Record<string, any>): boolean {
  if (!attributes || Object.keys(attributes).length === 0) {
    return false;
  }
}
```

If it is an array:

```js
function isTomster(attributes?: Array<string>): boolean {
  if (!attributes || attributes.length === 0) {
    return false;
  }
}
```

Lastly, the guides can also point users to a utility library like [lodash](https://lodash.com/docs).

For many existing codebases, it should be as simple as installing the extracted addon and running the codemod.  However, users not utilizing these methods will not have these methods bundled by default.

## Deprecation Guide

### compare

Install and import `ember-util` to `compare` two types.
until: 5.0.0
id: use-ember-util-package--compare

```js
import { compare } from '@ember/util';

compare(['code'], ['code', 'is', 'lovely']); // -1
compare(['code'], ['code']); // 0
compare(['code'], []); // 1
```

To avoid the `compare` deprecation, you can install the `ember-util` package and do the following

```js
import { compare } from 'ember-util';

compare(['code'], ['code', 'is', 'lovely']); // -1
compare(['code'], ['code']); // 0
compare(['code'], []); // 1
```

### isBlank

Install and import `ember-util` to check `isBlank` status.
until: 5.0.0
id: use-ember-util-package--isBlank

```js
import { isBlank } from '@ember/util';

isBlank(['code']); // false
isBlank([]); // true
```

To avoid the `isBlank` deprecation, you can install the `ember-util` package and do the following

```js
import { isBlank } from 'ember-util';

isBlank(['code']); // false
isBlank([]); // true
```

### isEmpty

Install and import `ember-util` to check `isEmpty` status.
until: 5.0.0
id: use-ember-util-package--isEmpty

```js
import { isEmpty } from '@ember/util';

isEmpty(['code']); // false
isEmpty([]); // true
```

To avoid the `isEmpty` deprecation, you can install the `ember-util` package and do the following

```js
import { isEmpty } from 'ember-util';

isEmpty(['code']); // false
isEmpty([]); // true
```

### isEqual

Install and import `ember-util` to check `isEqual` status.
until: 5.0.0
id: use-ember-util-package--isEqual

```js
import { isEqual } from '@ember/util';

isEqual({}, { a: 'key' }); // false
isEqual({ a: 'key' }, { a: 'key' }); // true
isEqual(['code'], ['code']); // false
```

To avoid the `isEqual` deprecation, you can install the `ember-util` package and do the following

```js
import { isEqual } from 'ember-util';

isEqual({}, { a: 'key' }); // false
isEqual({ a: 'key' }, { a: 'key' }); // true
isEqual(['code'], ['code']); // false
```

### isNone

Install and import `ember-util` to check `isNone` status.
until: 5.0.0
id: use-ember-util-package--isNone

```js
import { isNone } from '@ember/util';

isNone(undefined); // true
isNone([]); // false
isNone(''); // false
```

To avoid the `isNone` deprecation, you can install the `ember-util` package and do the following

```js
import { isNone } from 'ember-util';

isNone(undefined); // true
isNone([]); // false
isNone(''); // false
```

### isPresent

Install and import `ember-util` to check `isPresent` status.
until: 5.0.0
id: use-ember-util-package--isPresent

```js
import { isPresent } from '@ember/util';

isPresent(undefined); // false
isPresent([]); // false
isPresent('hi der'); // true
```

To avoid the `isPresent` deprecation, you can install the `ember-util` package and do the following

```js
import { isPresent } from 'ember-util';

isPresent(undefined); // false
isPresent([]); // false
isPresent('hi der'); // true
```

### typeOf

Install and import `ember-util` to check `typeOf` on an input.
until: 5.0.0
id: use-ember-util-package--typeOf

```js
import { typeOf } from '@ember/util';

let noop = () => {};
typeOf(noop); // 'function'
```

To avoid the `typeOf` deprecation, you can install the `ember-util` package and do the following

```js
import { typeOf } from 'ember-util';

let noop = () => {};
typeOf(noop); // 'function'
```

### notEmpty

Install and import `ember-util` to check `notEmpty` on a property.
until: 5.0.0
id: use-ember-util-package--notEmpty

```js
import { notEmpty } from '@ember/object/computed';

class ToDoList {
  constructor(todos) {
    set(this, 'todos', todos);
  }

  @notEmpty('args.todos') isDone;
}
```

To avoid the `notEmpty` deprecation, you can install the `ember-util` package and do the following

```js
import { notEmpty } from 'ember-util';

class ToDoList {
  constructor(todos) {
    set(this, 'todos', todos);
  }

  @notEmpty('todos') isDone;
}
```

### empty

Install and import `ember-util` to check `empty` on a property.
until: 5.0.0
id: use-ember-util-package--empty

```js
import { empty } from '@ember/object/computed';

class ToDoList {
  constructor(todos) {
    set(this, 'todos', todos);
  }

  @empty('todos') isDone;
}
```

To avoid the `empty` deprecation, you can install the `ember-util` package and do the following

```js
import { empty } from 'ember-util';

class ToDoList {
  constructor(todos) {
    set(this, 'todos', todos);
  }

  @empty('todos') isDone;
}
```

### none

Install and import `ember-util` to check `none` on a property.
until: 5.0.0
id: use-ember-util-package--none

```js
import { none } from '@ember/object/computed';

class ToDoList {
  @none('args.todos') isDone;
}
```

To avoid the `none` deprecation, you can install the `ember-util` package and do the following

```js
import { none } from 'ember-util';

class ToDoList {
  @none('args.todos') isDone;
}
```

## Drawbacks

This change may impact quite a few codebases.  Moreover, users who heavily rely on these utility methods will have to install another addon.  Some users may desire this functionality out of the box.

Given the example above, some users see a lot of value for the safety checks `isPresent(myVariable)` provides.  The type of variable could be of type String or of type Object and wiring this up to check manually is laborious.

## Alternatives

Another option would be to improve existing utility cases such as `isEmpty({}) === true`.  Moreover, we could also consider scoping existing utility methods to specific types. For example, `isEmpty` would be specific to checking objects, collections, maps or sets to reduce confusion of when these utility methods should be used.

Instead of deprecating the `notEmpty`, `empty`, `none` computed macros one-by-one, it would probably be good to deprecate the entire `@ember/object/computed` module because everything else in there is not aligned with current idioms.

## Unresolved questions

