---
Start Date: 2018-09-10
RFC PR: https://github.com/emberjs/rfcs/pull/373
Ember Issue: (leave this empty)

---

# Element Modifier Manager

## Summary

This RFC proposes a low-level primitive for defining element modifiers. It is a parent to the [Modifiers RFC](https://github.com/emberjs/rfcs/pull/353).

## Motivation

Ever since Ember 1.0 we have had the concept of element modifiers, however Ember only exposes one modifier; `{{action}}`. We also do not provide a mechanism for defining your own modifiers and managing their life cycles.

As [pointed out](https://github.com/emberjs/rfcs/pull/353#issuecomment-417769349) in the [Element Modifiers RFC](https://github.com/emberjs/rfcs/pull/353) we should expose the underlying infrastructure that makes element modifiers possible. Based on our experience, we believe it would be beneficial to open up these new primitives to the wider community. The largest benefit is that it allows the community to experiment with and iterate on APIs outside of the core framework.

This RFC is in the same spirit as the [custom components RFC](https://github.com/emberjs/rfcs/blob/master/text/0213-custom-components.md).

## Detailed design

This RFC introduces the concept of _modifier managers_. A modifier manager is an object that is responsible for coordinating the lifecycle events that occurs when invoking, installing and updating an element modifier.

### Registering modifier managers

Modifier managers are registered with the `modifier-manager` type in the
application's registry. Similar to services, modifier managers are singleton
objects (i.e. `{ singleton: true, instantiate: true }`), meaning that Ember
will create and maintain (at most) one instance of each unique modifier
manager for every application instance.

To register a modifier manager, an addon will put it inside its `app` tree:

```js
// ember-basic-component/app/modifier-managers/basic.js

import EmberObject from '@ember/object';

export default EmberObject.extend({
  // ...
});
```

(Typically, the convention is for addons to define classes like this in its
`addon` tree and then re-export them from the `app` tree. For brevity, we will
just inline them in the `app` tree directly for the examples in this RFC.)

This allows the modifier manager to participate in the DI system – receiving
injections, using services, etc. Alternatively, modifier managers can also
be registered with imperative API. This could be useful for testing or opt-ing
out of the DI system. For example:

```js
// ember-basic-modifier/app/initializers/register-basic-modifier-manager.js

const MANAGER = {
  // ...
};

export function initialize(application) {
  // We want to use a POJO here, so we are opt-ing out of instantiation
  application.register('modifier-manager:basic', MANAGER, { instantiate: false });
}

export default {
  name: 'register-basic-modifier-manager',
  initialize
};
```

## Determining which modifier manager to use

When invoking the modifier `<p {{foo baz bar=bar}} />`, Ember will first resolve the
modifier class (`modifier:foo`, usually the `default` export from
`app/modifiers/foo.js`). Next, it will determine the appropiate modifier
manager to use based on the resolved modifier class.

Ember will provide a new API to assign the modifier manager for a element modifier
class:


```js
// my-app/app/modifier/foo.js

import EmberObject from '@ember/object';
import { createManager } from './basic-manager';
import { setModifierManager } from '@ember/modifier';

export default setModifierManager(createManager, EmberObject.extend({
  // ...
}));
```

```js
// my-app/app/modifier/basic-manager.js

// ...

export function createManager(owner) {
  return new BasicManager(owner);
}
```

`setModifierManager` takes two parameters. The first parameter is a function that takes an `Owner` and returns an instance of a manager. The second parameter is the base class that applications would extend from.

In reality, an app developer would never have to write this in their apps,
since the modifier manager would already be assigned on a super-class provided
by the framework or an addon. The `setModifierManager` function is essentially
a low-level API designed for addon authors and not intended to be used by app
developers. Attempting to reassign the modifier manager when one is already
assinged on a super-class will be an error. If no modifier manager is set, it
will also result in a runtime error when invoking the modifier.

## Modifier Lifecycle

Back to the `<p {{foo baz bar=bar}}></p>` example.

Once Ember has determined the modifier manager to use, it will be used to manage the modifiers's lifecycle.

### `createModifier`

The first step is to create an instance of the modifier. Ember will invoke the modifier manager's `createModifier` method:

```js
// ember-basic-component/app/modifier-managers/basic.js

import EmberObject from '@ember/object';

export default EmberObject.extend({
  createModifier(factory, args) {
    return factory.create(args);
  },
});
```

The `createModifier` method on the modifier manager is responsible for taking the modifier's factory and the arguments passed to the modifier (the ... in {{foo ...}}) and return an instantiated modifier.

The first argument passed to `createModifier` is the result returned from the `factoryFor` API. It contains a class property, which gives you the the raw class (the default export from app/modifiers/foo.js) and a create function that can be used to instantiate the class with any registered injections, merging them with any additional properties that are passed.

The second argument is a snapshot of the arguments passed to the modifier in the template invocation, given in the following format:

```js
{
  positional: [ ... ],
  named: { ... }
}
```

For example, given the following invocation:

```hbs
<p {{foo baz bar=bar}}></p>
```

You will get the following as the second argument:

```js
{
  positional: [true],
  named: {
    "bar": "Another RFC by Chad"
  }
}
```

The arguments object should not be mutated (e.g. args.positional.pop() is no good). In development mode, it might be sealed/frozen to help prevent these kind of mistakes.

This hook has the following timing semantics:

**Always**
- called as discovered during DOM construction
- called in defintion order in template

### `installModifier`

Once the modifier instance has been created, the next step is to install the modifier on to the underlying element.

```js
// ember-basic-component/app/modifier-managers/basic.js

import EmberObject from '@ember/object';

export default EmberObject.extend({
  createModifier(factory, args) {
    return factory.create(args.named);
  },

  installModifier(instance, element, args) {
    instance.element = element;
    if (instance.wasInstalled !== undefined) {
      instance.wasInstalled(args.positional, args.named);
    }
  },

  // ...
});
```

`installModifer` is responsible for giving access to the underlying element and arguments to the modifier instance.

The first argument passed to `installModifer` is the result of `createModifier`. The second argument is the `element` the modifier was defined on. The third argument is the same snapshot of the arguments passed to the modifier in the template invocation that `createModifier` recieved.

This hook has the following timing semantics:

**Always**

- called after all children modifier managers `installModifer` hook are called
- called after DOM insertion

**May or May Not**

- be called in the same tick as DOM insertion
- have the sibling nodes fully initialized in DOM

### `updateModifier`

Modifiers are only updated when one of its arguments is changed. In this case Ember will call the manager's `updateModifier` method to give the manager the oppurtunity to reflect those changes on the modifier instance, before re-rendering.

```js
// ember-basic-component/app/modifier-managers/basic.js

import EmberObject from '@ember/object';

export default EmberObject.extend({
  createModifier(factory, args) {
    return factory.create(args.named);
  },

  installModifier(instance, element, args) {
    instance.element = element;
    if (instance.wasInstalled !== undefined) {
      instance.wasInstalled(args.positional, args.named);
    }
  },

  updateModifier(instance, args) {
    if (instance.didUpdateArguments !== undefined) {
      instance.didUpdateArguments(args.positional, args.named);
    }
  }

  // ...
});
```

`updateModifier` recieves the modifier instance and also the the updated snapshot of arguments.

This hook has the following timing semantics:

**Always**

- called after the arguments to the modifier have changed

**Never**

- called if the arguments to the modifier are constants

### `destroyModifier`

`destroyModifier` will be called when the modifier is no longer needed. This is intended for performing object-model level cleanup.

```js
// ember-basic-component/app/modifier-managers/basic.js

import EmberObject from '@ember/object';

export default EmberObject.extend({
  createModifier(factory, args) {
    return factory.create(args.named);
  },

  installModifier(instance, element, args) {
    instance.element = element;
    if (instance.wasInstalled !== undefined) {
      instance.wasInstalled(args.positional, args.named);
    }
  },

  destroyModifier(instance, args) {
    if (instance.willDestroyDOM !== undefined) {
      instance.willDestroyDOM();
    }
  }
});
```

This hook has the following timing semantics:

**Always**

- called after all children modifier manager's `destroyModifier` hook is called

**May or May Not**

- be called in the same tick as DOM removal

## Capabilities

In addition to the methods specified above, modifier managers are required to
have a `capabilities` property.  This property must be set to the result of
calling the `capabilities` function provided by Ember.

### Versioning

The first, mandatory, argument to the `capabilities` function is the modifier
manager API, which is denoted in the `${major}.${minor}` format, matching the
minimum Ember version this manager is targeting. For example:

```js
// ember-basic-component/app/component-managers/basic.js

import { capabilities } from '@ember/modifier';
import EmberObject from '@ember/object';

export default EmberObject.extend({
  capabilities: capabilities('3.6'),

  createModifier(factory, args) {
    return factory.create(args.named);
  },

  installModifier(instance, element, args) {
    instance.element = element;
    if (instance.wasInstalled !== undefined) {
      instance.wasInstalled(args.positional, args.named);
    }
  },

  destroyModifier(instance, args) {
    if (instance.willDestroyDOM !== undefined) {
      instance.willDestroyDOM();
    }
  }
});
```

This allows Ember to introduce new capabilities and make improvements to this
API without breaking existing code.

Here is a hypothical scenario for such a change:

1. Ember 3.6 implemented and shipped the modifier manager API as described in
   this RFC.

2. The `ember-basic-modifier` addon released version 1.0 with the modifier
   manager shown above (notably, it declared `capabilities('3.6')`).

3. In Ember 3.8, we determined that constructing the arguments object passed to
   the hooks is a major performance bottleneck, and changes the API to pass a
   "proxy" object with getter methods instead (e.g. `args.getPositional(0)` and
   `args.getNamed('foo')`).

   However, since Ember sees that the `basic` modifier manager is written to
   target the `3.6` API version, it will retain the old behavior and passes the
   old (more expensive) "reified" arguments object instead, to avoid breakage.

4. The `ember-basic-modifier` addon author would like to take advantage of
   this performance optimization, so it updates its modifier manager code to
   work with the arguments proxy and changes its capabilities declaration to
   `capabilities('3.8')` in version 2.0.

This system allows us to rapidly improve the API and take advantage of
underlying rendering engine features as soon as they become available.

Note that addon authors are not _required_ to update to the newer API.
Concretely, modifier manager APIs have the following support policy:

* API versions will continue to be supported in the same major release of
  Ember. As shown in the example above, `ember-basic-modifier` 1.0 (which
  targets modifier manager API version 3.6), will continue to work on
  Ember 3.8. However, the reverse is not true – modifier manager API version
  3.8 will (somewhat obviously) not work in Ember 3.6.

* In addition, to ensure a smooth transition path for addon authors and app
  developers across major releases, each Ember version will support (at least)
  the previous LTS version as of the release was made. For example, if 3.16 is
  the last LTS release of the 3.x series, the modifier manager API version
  3.16 will be supported by Ember 4.0 through 4.4, at minimum.

Addon authors can also choose to target multiple versions of the modifier
manager API using [ember-compatibility-helpers](https://github.com/pzuraq/ember-compatibility-helpers/):

```js
// ember-basic-modifier/app/modifier-managers/basic.js

import { gte } from 'ember-compatibility-helpers';

let ComponentManager;

if (gte('3.5')) {
  ComponentManager = EmberObject.extend({
    capabilities: capabilities('3.8'),

    // ...
  });
} else {
  ComponentManager = EmberObject.extend({
    capabilities: capabilities('3.6'),

    // ...
  });
}

export default ComponentManager;
```

Since the conditionals are resolved at build time, the irrevelant code will be
stripped from production builds, avoiding any deprecation warnings.

### Optional Features

The second, optional, argument to the `capabilities` function is an object
enumerating the optional features requested by the modifier manager.

In the hypothical example above, while the "reified" arguments objects may be
a little slower, they are certainly easier to work with, and the performance
may not matter to but the most performance critical modifiers. A modifier
manager written for Ember 3.8 (again, only hypothically) and above would be
able to explicitly opt back into the old behavior like so:

```js
// ember-basic-component/app/component-managers/basic.js

import { capabilities } from '@ember/component';
import EmberObject from '@ember/object';

export default EmberObject.extend({
  capabilities: capabilities('3.8', {
    reifyArguments: true
  }),

  // ...
});
```

In general, we will aim to have the defaults set to as bare-bone as possible,
and allow the component managers to opt into the features they need in a PAYGO
(pay-as-you-go) manner, which aligns with the Glimmer VM philosophy. As the
rendering engine evolves, more and more feature will become optional.

At this time this RFC does not specify any optional capabilties for the initial release.

## How we teach this

What is proposed in this RFC is a low-level primitive. We do not expect most users to interact with this layer directly. Instead, most users will simply benefit from this feature by subclassing these special modifiers provided by addons.

## Drawbacks

In the long term, there is a risk of fragmentating the Ember ecosystem with many competing modifier APIs. However, given the Ember community's strong desire for conventions, this seems unlikely. We expect this to play out similar to the data-persistence story – there will be a primary way to do things (Ember Data), but there are also plenty of other alternatives catering to niche use cases that are underserved by Ember Data.

## Alternatives
Instead of focusing on exposing enough low-level primitives we can just ship the high level API as described in [RFC#353](https://github.com/emberjs/rfcs/pull/353).
## Unresolved questions

TBD?
