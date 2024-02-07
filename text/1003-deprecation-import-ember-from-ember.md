---
stage: accepted
start-date: 2024-01-22T00:00:00.000Z
release-date:
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1003 
project-link:
---

<!---
Directions for above:

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
-->

<-- Replace "RFC title" with the title of your RFC -->
# Deprecate `import Ember from 'ember'; 

## Summary

This RFC proprosing deprecating all APIs that have module-based replacements, as described in [RFC #176](https://rfcs.emberjs.com/id/0176-javascript-module-api) as well as other `Ember.*` apis that are no longer needed.

## Motivation

The `import Ember from 'ember';` set of APIs is implementn as a barrel file, and properly optimizing barrel files [is a lot of work, requiring integration with build time tools](https://vercel.com/blog/how-we-optimized-package-imports-in-next-js).

By removing this set of exports, we have an opportunity to shrink some apps (as some APIs are not used), improving the load performance of ember apps -- and we removing all of these gives us a chance to have a better grasp of what we can get rid of permananently.

Many of these APIs already have alternatives, and those will be called out explicitly in the _Transition Path_ below.

## Transition Path

This list is semi-exhaustive, in that it covers _every_ export from 'ember', but may not exhaustivily provide alternatives.

### `Ember.isNamespace`

No replacement, not needed.

### `Ember.toString`

No replacement, not needed.
e
### `Ember.Container`

No replacement. Both value and type are not needed.

### `Ember.Registry` 

No replacement. Both value and type are not needed.
Every property/method on this value is private.

### `Ember.testing`

Instead, use

```js
import { macroCondition, isTesting } from '@embroider/macros';

// ...

if (macroCondition(isTesting()) {
  // test only code here
}
```

### `Ember._setComponentManager`

Instead, use 
```js
import { setComponentManager } from '@ember/component';
```

### `Ember._componentManagerCapabilities`

Instead, use 
```js
import { capabilities } from '@ember/component';
```

### `Ember._modifierManagerCapabilities`

Instead, use 
```js
import { capabilities } from '@ember/modifier';
```

### `Ember.meta`

No replacement, not public API.
But meta may be used for the ember-inspector, but there are discussions starting around the time of this RFC to rework how ember-source and the inspector communicate with each other.

### `Ember._createCache`

Instead, use

```js
import { createCache } from '@glimmer/tracking/primitives/cache';
```

Implemented from [RFC#615](https://rfcs.emberjs.com/id/0615-autotracking-memoization)

### `Ember._cacheGetValue`

Instead, use 
```js
import { getValue } from '@glimmer/tracking/primitives/cache';
```

Implemented from [RFC#615](https://rfcs.emberjs.com/id/0615-autotracking-memoization)

### `Ember._cacheIsConst`

Instead, use 
```js
import { isConst } from '@glimmer/tracking/primitives/cache';
```

Implemented from [RFC#615](https://rfcs.emberjs.com/id/0615-autotracking-memoization)

### `Ember._descriptor`

No replacement. Internal utility for helping author decorators. 

### `Ember._getPath`

No replacement. Used for deep getting a property on an object unless it's destroyed.

### `Ember._setClassicDecorator`

No replacement. Internal utility for helping author decorators. 

### `Ember._tracked`

Instead, use

```js
import { tracked } from '@glimmer/tracking';
```

### `Ember.beginPropertyChanges`

No replacement.

### `Ember.changeProperties`

No replacement.

### `Ember.endPropertyChanges`

No replacement.

### `Ember.hasListeners`

No replacement.

### `Ember.libraries`

No replacement. 

App authors could choose to use any webpack or other build plugin that collections this information, such as [webpack-node-modules-list](https://github.com/ubilabs/webpack-node-modules-list). This additionally means that V1 libraries that pushed themselves into `Ember.libraries` no longer need to worry about interacting with this or any similar API. 


### `Ember._ContainerProxyMixin

No replacement. Mixins have been recommended against since Octane's release. 

### `Ember._ProxyMixin`

No replacement. Mixins have been recommended against since Octane's release. 

### `Ember._RegistryProxyMixin`

No replacement. Mixins have been recommended against since Octane's release. 

### `Ember.ActionHandler`

No replacement.

### `Ember.Comparable`

No replacement. Mixins have been recommended against since Octane's release. 

### `Ember.RSVP`

Instead use
```js
import RSVP from 'rsvp';
```

### `Ember._Cache`

No replacement. A general `Map`-based Cache that tracks the misses and hits.

### `Ember.GUID_KEY`

No replacement. Private.

### `Ember.canInvoke`

Instead use [optional chaining](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining):
```js
this.foo?.method?.();
```
### `Ember.generateGuid`

No replacement. Private.

### `Ember.guidFor`

Instead use
```js
import { guidFor } from '@ember/object/internals';
```

### `Ember.uuid`

No replacement. 

### `Ember.wrap`

No replacement. Private.

### `Ember.getOwner`

Instead, use 
```js
import { getOwner } from '@ember/owner';
```

### `Ember.onLoad`

Instead, use 
```js
import { onLoad } from '@ember/application';
```

### `Ember.runLoadHooks`

Instead, use 
```js
import { runLoadHooks } from '@ember/application';
```

### `Ember.setOwner`

Instead, use 
```js
import { setOwner } from '@ember/owner';
```

### `Ember.Application`

Instead, use 
```js
import Application from '@ember/application';
```

### `Ember.ApplicationInstance`

Instead, use 
```js
import ApplicationInstance from '@ember/application/instance';
```

### `Ember.Namespace`

Instead, use
```js
import Namespace from '@ember/application/namespace';
```

### `Ember.A`

Instead, use
```js
import { A } from '@ember/array';
```

### `Ember.Array`

Instead, use
```js
import EmberArray from '@ember/array';
```

### `Ember.NativeArray`

Instead, use
```js
import { NativeArray } from '@ember/array';
```

### `Ember.isArray`

Instead, use
```js
import { isArray } from '@ember/array';
```

### `Ember.makeArray`

Instead, use
```js
import { makeArray } from '@ember/array';
```

### `Ember.MutableArray`

Instead, use
```js
import MutableArray from '@ember/array/mutable';
```

### `Ember.ArrayProxy`

Instead, use
```js
import ArrayProxy from '@ember/array/proxy';
```

### `Ember.FEATURES`

No replacement.
But this is useful when working with feature flags.
This information could live on a specially named `globalThis` property, the enabled features could be emitted as a virtual module to import.

### `Ember._Input`

Instead, use
```js
import { Input } from '@ember/component';
```

### `Ember.Component`

Instead, use
```js
import Component from '@ember/component';
```

### `Ember.Helper`

Instead, use
```js
import Helper from '@ember/component/helper';
```

### `Ember.Controller`

Instead, use
```js
import Controller from '@ember/controller';
```

### `Ember.ControllerMixin`

No replacement.

### `Ember._captureRenderTree`

No replacement. But used by ember-inspector.

### `Ember.assert`

Instead, use
```js
import { assert } from '@ember/debug';
```

### `Ember.warn

Instead, use
```js
import { warn } from '@ember/debug';
```

### `Ember.debug`

Instead, use
```js
import { debug } from '@ember/debug';
```
### `Ember.deprecate`

Instead, use
```js
import { deprecate } from '@ember/debug';
```

### `Ember.deprecateFunc`

No replacement. Private.

### `Ember.runInDebug`

Instead, use
```js
import { runInDebug } from '@ember/debug';
```

### `Ember.inspect`

No replacement. Private.

### `Ember.Debug`

No replacement.

#### `Ember.Debug.registerDeprecationHandler`

Instead, use
```js
import { registerDeprecationHandler } from '@ember/debug';
```

### `Ember.ContainerDebugAdapter`

Instead, use
```js
import ContainerDebugAdapter from '@ember/debug/container-debug-adapter';
```

### `Ember.DataAdapter`

Instead, use
```js
import DataAdapter from '@ember/debug/data-adapter';
```

### `Ember._assertDestroyablesDestroyed`

Instead, use
```js
import { assertDestroyablesDestroyed } from '@ember/destroyable';
```

### `Ember._associateDestroyableChild`

Instead, use
```js
import { associateDestroyableChild } from '@ember/destroyable';
```

### `Ember._enableDestroyableTracking`

Instead, use
```js
import { enableDestroyableTracking } from '@ember/destroyable';
```

### `Ember._isDestroying`

Instead, use
```js
import { isDestroying } from '@ember/destroyable';
```

### `Ember._isDestroyed`

Instead, use
```js
import { isDestroyed } from '@ember/destroyable';
```

### `Ember._registerDestructor`

Instead, use
```js
import { registerDestructor } from '@ember/destroyable';
```

### `Ember._unregisterDestructor`

Instead, use
```js
import { unregisterDestructor } from '@ember/destroyable';
```

### `Ember.destroy`

Instead, use
```js
import { destroy } from '@ember/destroyable';
```

### `Ember.Engine`
### `Ember.EngineInstance`
### `Ember.Enumerable`
### `Ember.MutableEnumerable`
### `Ember.instrument`
### `Ember.subscribe`
### `Ember.Instrumentation`
### `Ember.Object`
### `Ember._action`
### `Ember.computed`
### `Ember.defineProperty`
### `Ember.get`
### `Ember.getProperties`
### `Ember.notifyPropertyChange`
### `Ember.observer`
### `Ember.set`
### `Ember.trySet`
### `Ember.setProperties`
### `Ember.cacheFor`
### `Ember._dependentKeyCompat`
### `Ember.ComputedProperty`
### `Ember.expandProperties`
### `Ember.CoreObject`
### `Ember.Evented`
### `Ember.on`
### `Ember.addListener`
### `Ember.removeListener`
### `Ember.sendEvent`
### `Ember.Mixin`
### `Ember.mixin`
### `Ember.Observable`
### `Ember.addObserver`
### `Ember.removeObserver`
### `Ember.PromiseProxyMixin`
### `Ember.ObjectProxy`
### `Ember.RouterDSL`
### `Ember.controllerFor`
### `Ember.generateController`
### `Ember.generateControllerFactory`
### `Ember.generateControllerFactory`
### `Ember.HashLocation`
### `Ember.HistoryLocation`
### `Ember.NonLocation`
### `Ember.Route`
### `Ember.Router`
### `Ember.run`
### `Ember.Service`
### `Ember.compare`
### `Ember.isBlank`
### `Ember.isEmpty`
### `Ember.isEqual`
### `Ember.isPresent`
### `Ember.typeOf`
### `Ember.VERSION`
### `Ember.ViewUtils`
### `Ember._getComponentTemplate`
### `Ember._helperManagerCapabilities`
### `Ember._setComponentTemplate`
### `Ember._setHelperManager`
### `Ember._setModifierManager`
### `Ember._templateOnlyComponent`
### `Ember._invokeHelper`
### `Ember._hash`
### `Ember._array`
### `Ember._concat`
### `Ember._get`
### `Ember._on`
### `Ember._fn`
### `Ember._Backburner`
### `Ember.inject`
### `Ember.__loader`
### `Ember.ENV`

Use
```
import MyEnv from '<my-app>/config/environment';
```

### `Ember.lookup`

### `Ember.onerror`

Use
```js
window.addEventListener('error', /* ... event handler ... */);
```

### `Ember.testing`
### `Ember.BOOTED`
No replacement. Unused.
### `Ember.TEMPLATES`

This is the template registry. It does not need to be public API. 

No replacement.

### `Ember.HTMLBars`

No replacement.

### `Ember.Handlebars`

No replacement.

#### `Ember.Handlebars.Utils.escapeExpression`

Removed in [ember.js PR#20360](https://github.com/emberjs/ember.js/pull/20360) as it is not public API.


### `Ember.Test`
### `Ember.setupForTesting`

## How We Teach This

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?
Does it mean we need to put effort into highlighting the replacement
functionality more? What should we do about documentation, in the guides
related to this feature?
How should this deprecation be introduced and explained to existing Ember
users?

> Keep in mind the variety of learning materials: API docs, guides, blog posts, tutorials, etc.

Available Codemods

- https://github.com/ember-codemods/ember-modules-codemod (from the work of RFC 176)

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.
There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

As a migration path for folks, we could add a package.json#exports entry to [`ember`](https://www.npmjs.com/package/ember)
that re-exports the still-available APIs.

Some APIs we don't want or need to keep at all though.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
