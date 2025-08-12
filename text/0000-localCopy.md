---
stage: accepted
start-date: 2025-08-12T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams:
  - framework
  - typescript
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
suite: 
---

# Add `localCopy` reactive primitive

## Summary

This RFC proposes adding `localCopy` and `@localCopy` to Ember's reactive primitives, providing a way to create local, mutable copies of reactive state that maintain synchronization with their source while allowing local and independent updates.

## Motivation

Modern reactive applications often need to create local, editable copies of remote or shared state. Common use cases include:

1. Controlled form inputs can maintain local tracked state until form submission (submit commonly re-synchronizes the source of truth)
3. Allowing users to make changes that can be saved or discarded
3. Preventing child components from accidentally mutating parent state

Currently, developers must manually implement this pattern using multiple `@tracked` properties and bespoke synchronization logic. This leads to boilerplate code and potential bugs around state consistency.

`@localCopy` has existed for a long time in a library, [`tracked-toolbox`](https://github.com/tracked-tools/tracked-toolbox) and has proven its value -- just as `@cached` did (also originally defined in `tracked-toolbox` (and is still exported from there)).

`@localCopy` solves all of the above as well as any other use case for "forking tracked state".


## Detailed design

The test-suite of `@localCopy` from [`tracked-toolbox`](https://github.com/tracked-tools/tracked-toolbox/blob/master/test-app/tests/unit/local-copy-test.js) as well as that of []`localCopy`](https://github.com/proposal-signals/signal-utils/blob/main/tests/local-copy.test.ts) and []`@localCopy`](https://github.com/proposal-signals/signal-utils/blob/main/tests/%40localCopy.test.ts) from `signal-utils` describe the acceptance criteria for this utility.

### API Overview

The `localCopy` export is available in both function and decorator forms:

```typescript
import { localCopy } from '@ember/reactive';
```

Because function usage and decorator usage have different arity, we can utilize the same export for all styles of usage.

### Function Form

The function form creates a reactive primitive that implements the same interface as a [Cell](https://github.com/emberjs/rfcs/pull/1071):

```typescript
function localCopy<T>(
  source: () => T,
  options?: {
    equals?: (a: T, b: T) => boolean;
    description?: string;
  }
): Cell<T>;
```

See [Cell](https://github.com/emberjs/rfcs/pull/1071/files#diff-fa519f723fb6a105edfe2779ca6e4593bce756817da177495468021b37c46f3eR114).

#### Basic Usage

```typescript
import { tracked } from '@glimmer/tracking';
import { localCopy } from '@ember/reactive';

class FormComponent extends Component {
  @tracked user = { name: 'John', email: 'john@example.com' };
  
  /**
   * second argument is optional
   */
  userCopy = localCopy(() => this.user, {
    equals: (a, b) => a.name === b.name && a.email === b.email
  });
  
  @action
  updateName(newName) {
    // Updates the local copy without affecting the original (this.user)
    this.userCopy.update(user => ({ ...user, name: newName }));
    // or
    this.userCopy.current = { name: 'John', email: 'foo@foo.com' };
  }
  
  @action
  save() {
    this.user = this.userCopy.current;
  }
}
```

### Decorator Form

The decorator form provides convenient syntax for class properties:

```glimmer-ts
import { localCopy } from '@ember/reactive';

class EditableProfile extends Component {
  @tracked profile = { name: 'Jane', bio: 'Developer' };
  
  /**
   * second argument is optional.
   * first argument must be a string because the left-hand side of a property does not have access to the instance.
   */
  @localCopy('profile', { 
    equals: (a, b) => a.name === b.name && a.bio === b.bio 
  }) profileCopy;
  
  updateBio(newBio) {
    this.profileCopy = { name: 'Jane', bio: 'Vlogger' };
  }

  <template>
    Bio: {{this.profileCopy.bio}}
    Name: {{this.profileCopy.name}}

    <button {{on 'click' this.updateBio}}>Update</button>
  </template>
}
```

### Cell Interface Compatibility

The function form of `localCopy` implements the same interface as a [Cell](https://github.com/emberjs/rfcs/pull/1071), making it interoperable with other Cell-based APIs:

```typescript
// Both have the same core interface
const cell = cell(initialValue);
const copy = localCopy(() => sourceValue);

// All of these work with both:
cell.current = newValue;
copy.current = newValue;

cell.set(newValue);
copy.set(newValue);

cell.update(fn => fn(current));
copy.update(fn => fn(current));

cell.freeze();
copy.freeze();
```

This compatibility ensures `localCopy` can be used anywhere a Cell is expected and the decorator usage matches the API of `@tracked`, enabling composition with all existing concepts.


## How we teach this

### Conceptual Introduction

`localCopy` should be introduced as a "reactive copy" or "state forking" primitive that solves the common pattern of wanting to edit data without immediately affecting the original.

Potential guides content from the tracked-toolbox README:


`@localCopy` Creates a local copy of a remote value. The local copy can be updated locally,
but will also update if the remote value ever changes:

```js
import Component from '@glimmer/component';
import { localCopy } from '@ember/reactive';

export default class CustomInput extends Component {
  // This defaults to the value of this.args.text
  @localCopy('args.text') text;

  updateText({ target }) {
    // this updates the value of `text`, but does _not_ update
    // the value of `this.args.text`
    this.text = String(target.value);

    this.args.onInput?.(this.text);
  }

  <template>
    <input {{on 'input' this.updateText}}>
    ...
  </template>
}
```

In this example, if `args.text` were to ever change externally, then the local
`text` property would also update. The local copy is not a clone of the value
passed in, it is the actual value itself, so values like arrays and objects
will still be affected upstream if their values are changed.

An initializer can be provided as the second parameter to the decorator. This
will be used if the remote value is `undefined`:

```js
export default class CustomInput extends Component {
  @localCopy('args.text') text;
}
```

If the initializer is a function, it will be called and its return value will be
used as the default value.


### Differences from tracked-toolbox

- localCopy's second argument is _not_ a "placeholder"
  to use a placeholder, or fallback value, folks would want to use a getter:
  ```js
  get textWithDefault() {
    return this.text ?? 'default value';
  }
  ```

## Drawbacks

- Adds another reactive tool to learn and understand
   (tracked-toolbox is already very popular though)

## Alternatives

- continue using tracked-toolbox

## Unresolved questions

n/a
