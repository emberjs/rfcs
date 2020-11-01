---
Start Date: 2018-10-23
RFC PR: https://github.com/emberjs/rfcs/pull/392
Ember Issue: (leave this empty)

---

# Summary

This deprecates the string-based lookup API for associating a custom component manager with a corresponding base class.

```js
import EmberObject from '@ember/object';
import { setComponentManager } from '@ember/component';

export default setComponentManager('basic', EmberObject.extend({
  //...
}))
```

Instead, you must pass a factory function that produces an instance of the custom manager:

```js
import EmberObject from '@ember/object';
import { createManager } from './basic-manager';
import { setComponentManager } from '@ember/modifier';

export default setComponentManager(createManager, EmberObject.extend({
  // ...
}));
```

Where `createManager` is:

```js
export function createManager(owner) {
  return new BasicManager(owner);
}
```

# Motivation

There are several motivators:

- A string-based API is not friendly when it comes to tree shaking. It would force us into creating a compiler to turn the string into a symbol that build tools like Rollup and Webpack could analyze.
- This API expands the namespacing problems associated with module unification. Specifically, an addon author would have to associate the package name with the string, similar to how [RFC#367](https://github.com/mixonic/rfcs/blob/mu-packages/text/0000-module-unification-packages.md#explicit-packages-for-service-injections) proposes changes to services.
- `setModifierManager` as introduced by [RFC#373](https://github.com/emberjs/rfcs/blob/89349d30ade24303a06448bc121b8fd810cbe58d/text/0373-Element-Modifier-Managers.md#determining-which-modifier-manager-to-use) uses a factory style API. This RFC intends to align these to function signatures.
- We want to make sure this API is compatible with the binary AoT compilation work we did in the Glimmer-VM.

# Transition Path

We can transition away by producing a factory function in the internals of `setupComponentManager`. The implementation would look something like:

```js
export function setComponentManager(stringOrFunction, obj: any) {
  let factory;
  if (typeof stringOrFunction === 'string') {
    deprecate(
      `Passing the name of the component manager to 'setupComponentManager' is deprecated. Please pass a function that produces an instance of the manager.`,
      {
        id: 'deprecate-string-based-component-manager',
        unil: '4.0.0'
      }
    );
    factory = function(owner: Owner) {
      return owner.lookup(`component-manager:${stringOrFunction}`);
    };
  } else {
    factory = stringOrFunction;
  }

  // ...
}

```

# How We Teach This

From our understanding this API has very limited usage as it is a low-level API. We should update that docs accordingly.

# Drawbacks

Historically, Ember has given developers base classes that the developer would extend from and Ember would create instances on your behalf. This allows the framework to know that the interface of the object is complete. With this approach we are relying more on the addon author to construct the object and ensure it conforms to the correct interface.

Generally speaking this a good practice of OOP but due to the fact that JavaScript does not have first-class interfaces, Ember has taken the concretion approach.

# Alternatives

Instead of passing a factory function we could pass the class itself. This option does have the issue of Ember needing to know how to construct the class and does not allow for the addon author to perform any dependency injections.

# Unresolved questions

TBD?
