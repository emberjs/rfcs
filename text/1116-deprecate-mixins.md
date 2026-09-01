---
stage: ready-for-release
start-date: 2025-06-20T00:00:00.000Z
release-date:
release-versions:
teams:
  - cli
  - framework
  - learning
  - typescript
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1116'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/1143'
project-link:
---

# Deprecating Mixin Support

## Summary

Deprecate all Mixin support.

## Motivation

For a while now, Ember has not recommended the use of Mixins. We should actually fully
deprecate this.

## Transition Path

All existing public Ember mixins will be deprecated:

- [Ember.Mixin](https://github.com/emberjs/rfcs/pull/1111) (from `@ember/object/mixin`)
- [Ember.PromiseProxyMixin](https://github.com/emberjs/rfcs/pull/1112) (from `@ember/object/promise-proxy-mixin`)
- [Ember.Comparable](https://github.com/emberjs/rfcs/pull/1113) (from `@ember/object/comparable`)
- [Ember.Enumerable](https://github.com/emberjs/rfcs/pull/1114) (from `@ember/object/enumerable`)
- [Ember.Observable](https://github.com/emberjs/rfcs/pull/1115) (from `@ember/object/observable`)

For users that still want mixin-like functionality, we should recommend class
decorators. If this feels less ergonimic that we would desire, we can consider
providing an addon that adds some sugar around this.

### Deprecation guides

The deprecation guides for each of the above are written out below, and are being
published to https://deprecations.emberjs.com/. Each guide targets `until: 8.0.0`.

#### `@ember/object/mixin` (since 7.4.0)

`Mixin` from `@ember/object/mixin` is deprecated. Mixins are part of the legacy `EmberObject` class system and do not work with native class syntax.

If your code still uses classic class syntax, convert it to native classes first, for example with the [ember-native-class-codemod](https://github.com/ember-codemods/ember-native-class-codemod). Then remove the mixins using one of the patterns below.

##### Before: sharing behavior with a mixin

```javascript
// app/mixins/editable.js
import Mixin from '@ember/object/mixin';

export default Mixin.create({
  isEditing: false,

  edit() {
    this.set('isEditing', true);
  },
});
```

```javascript
// app/models/comment.js
import EmberObject from '@ember/object';
import EditableMixin from '../mixins/editable';

export default class Comment extends EmberObject.extend(EditableMixin) {
  post = null;
}
```

##### After: sharing behavior with a subclass factory

A function that takes a base class and returns a subclass provides the same composition using only standard JavaScript:

```javascript
// app/utils/editable.js
import { tracked } from '@glimmer/tracking';

export function editable(Base) {
  return class extends Base {
    @tracked isEditing = false;

    edit() {
      this.isEditing = true;
    }
  };
}
```

```javascript
// app/models/comment.js
import EmberObject from '@ember/object';
import { editable } from '../utils/editable';

export default class Comment extends editable(EmberObject) {
  post = null;
}
```

Because the factory returns a class expression, it also works as a class decorator (`@editable`) if your build is configured for decorator syntax on classes.

##### Other replacements

Depending on what a mixin was doing, a different pattern may fit better:

- Shared state or behavior used across the app belongs in a service or service-like abstraction, injected where needed.
- Stateless helpers can be plain functions exported from a module (these are also easier to unit test).
- Behavior only used by one class hierarchy can move into a common base class (though composition is **strongly** recommended instead of inheritance).

#### `@ember/object/promise-proxy-mixin` (since 7.4.0)

`PromiseProxyMixin` from `@ember/object/promise-proxy-mixin` is deprecated. Use `async/await` with tracked state instead.

Before:

```gjs
import ObjectProxy from '@ember/object/proxy';
import PromiseProxyMixin from '@ember/object/promise-proxy-mixin';

const PromiseObject = ObjectProxy.extend(PromiseProxyMixin);

const proxy = PromiseObject.create({
  promise: fetchSettings(),
});

<template>
  {{#if proxy.isPending}}
    Loading...
  {{else if proxy.isFulfilled}}
    Value: {{proxy.content.value}}
  {{else if proxy.isRejected}}
    Error: {{proxy.reason}}
  {{/if}}
</template>
```

##### After: tracked properties and `async/await`

Manage the state of the asynchronous operation in the class that needs it:

```gjs
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';

export default class Settings extends Component {
  @tracked isLoading = true;
  @tracked error = null;
  @tracked content = null;

  constructor() {
    super(...arguments);
    this.loadData();
  }

  async loadData() {
    try {
      this.content = await fetchSettings();
    } catch (error) {
      this.error = error;
    } finally {
      this.isLoading = false;
    }
  }

  <template>
    {{#if this.isLoading}}
      Loading...
    {{else if this.error}}
      Error: {{this.error}}
    {{else}}
      Value: {{this.content.value}}
    {{/if}}
  </template>
}
```

##### After: an ember-concurrency task

For user-initiated or cancellable work, [ember-concurrency](https://ember-concurrency.com/docs/introduction) tasks expose the same lifecycle state that `PromiseProxyMixin` provided, along with cancellation and concurrency control:

```gjs
import Component from '@glimmer/component';
import { task } from 'ember-concurrency';

export default class Settings extends Component {
  load = () => void this.loadData.perform();

  loadData = task(async () => {
    return await fetchSettings();
  });

  <template>
    <button {{on "click" this.load}}>Load</button>

    {{#if this.loadData.isRunning}}
      Loading...
    {{else if this.loadData.last.error}}
      Error: {{this.loadData.last.error}}
    {{else if this.loadData.last.isSuccessful}}
      Value: {{this.loadData.last.value}}
    {{/if}}
  </template>
}
```

##### Migration strategy

Pick a replacement based on the use case:

1. For user-initiated async tasks (button clicks, form submissions), use ember-concurrency.
2. For data loading, consider [WarpDrive](https://warp-drive.io) (formerly Ember Data), whose `getRequestState` tracks request state for you (see also: [`getPromiseState`](https://reactive.nullvoxpopuli.com/functions/get-promise-state.getPromiseState.html) if not using WarpDrive).

#### `@ember/object/comparable` (since 7.2.0)

The `Comparable` mixin is deprecated.

Apps and addons should stop importing or extending `Comparable`. To provide custom comparison behavior, define a function-valued `compare(other)` method directly on the object or class instead.

Before:

```js
import EmberObject from '@ember/object';
import Comparable from '@ember/-internals/runtime/lib/mixins/comparable';

const Rectangle = EmberObject.extend(Comparable, {
  compare(other) {
    // custom comparison logic
  },
});
```

After:

```js
class Rectangle {
  // Returns -1, 0, 1
  compare(other) {
    // custom comparison logic
  },
}
```

Defining `compare(other)` directly is sufficient.

However, if you need to `sort` a list of your objects, you will want to define a separate [`comparator function`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort#description).

#### `@ember/enumerable` and `@ember/enumerable/mutable` (since 7.4.0)

`Enumerable` from `@ember/enumerable` and `MutableEnumerable` from `@ember/enumerable/mutable` are deprecated.

These mixins have been empty for a long time. The mixins were kept only so that existing `.detect()` checks kept working. They are now deprecated along with the rest of the mixin system.

##### Replacing `.detect()` checks

`Enumerable.detect(value)` answered "can I treat this value as a list?". Check for the capability you need instead.

Before:

```javascript
import Enumerable from '@ember/enumerable';

function printAll(maybeList) {
  if (Enumerable.detect(maybeList)) {
    maybeList.forEach((item) => console.log(item));
  }
}
```

After, for arrays:

```javascript
function printAll(maybeList) {
  if (Array.isArray(maybeList)) {
    maybeList.forEach((item) => console.log(item));
  }
}
```

Or, to accept any iterable (`Map`, `Set`, generators, and so on):

```javascript
function printAll(maybeList) {
  if (typeof maybeList?.[Symbol.iterator] === 'function') {
    for (let item of maybeList) {
      console.log(item);
    }
  }
}
```

##### Replacing custom enumerable classes

If you built a custom collection class with these mixins (usually through `EmberArray` or `MutableArray`), switch to a native array, or to `trackedArray` from [@ember/reactive/collections](https://api.emberjs.com/ember/release/modules/@ember%2Freactive%2Fcollections/) when the collection drives UI updates.

For collection classes with their own API, implement the [iterable protocol](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols) so consumers can use `for...of`, spread, and `Array.from`:

```javascript
class Queue {
  #items = [];

  add(item) {
    this.#items.push(item);
  }

  *[Symbol.iterator]() {
    yield* this.#items;
  }
}

let queue = new Queue();
queue.add('a');
queue.add('b');

[...queue]; // ['a', 'b']
```

The enumerable methods themselves (`firstObject`, `mapBy`, `pushObject`, and friends) live on `EmberArray` and `MutableArray`, which are deprecated as well. Replace them with native equivalents such as `arr[0]`, `map`, and `push`, using `trackedArray` where mutation needs to be tracked.

#### `@ember/object/observable` — `this.get` and `this.set` (since 7.4.0)

The `get` and `set` methods from `@ember/object/observable` are deprecated. This also applies to all built-in `Ember.Object` descendants.
To migrate, use native JavaScript getters and setters instead.

##### Replacing `.get()`

Instead of using `.get()`, use standard property access.

**Before**

```javascript
import EmberObject from '@ember/object';

class Person extends EmberObject {
  name = 'John Doe';
  details = {
    age: 30
  };
}

const person = new Person();

const name = person.get('name');
const age = person.get('details.age');
```

**After**

```javascript
class Person {
  name = 'John Doe';
  details = {
    age: 30
  };
}

const person = new Person();

const name = person.name;
const age = person.details.age;
```

For nested properties that might be null or undefined, use the optional chaining operator (`?.`):

```javascript
const street = person.address?.street;
```

##### Replacing `.set()`

Instead of using `.set()`, use standard property assignment.

**Before**

```javascript
import EmberObject from '@ember/object';

class Person extends EmberObject {
  name = 'John Doe';
}

const person = new Person();

person.set('name', 'Jane Doe');
```

**After**

```javascript
import { tracked } from '@glimmer/tracking';

class Person {
  @tracked name = 'John Doe';
}

const person = new Person();

person.name = 'Jane Doe';
```

## Exploration

To validate this deprecation, I've tried removing all Mixins in this PR: https://github.com/emberjs/ember.js/pull/20923

## How We Teach This

We should remove all references from the guides and publish the deprecation guides
written out in the Transition Path above. Those guides are tracked in these
deprecation-app PRs:

- `Mixin`: https://github.com/ember-learn/deprecation-app/pull/1434 (originally https://github.com/ember-learn/deprecation-app/pull/1408)
- `PromiseProxyMixin`: https://github.com/ember-learn/deprecation-app/pull/1435
- `Comparable`: https://github.com/ember-learn/deprecation-app/pull/1432
- `Enumerable`: https://github.com/ember-learn/deprecation-app/pull/1435
- `Observable`: https://github.com/ember-learn/deprecation-app/pull/1435

## Drawbacks

Some users probably rely on this functionality. However, it's almost certainly
something that we don't need to keep in Ember itself.

## Alternatives

None

## Unresolved questions

None
