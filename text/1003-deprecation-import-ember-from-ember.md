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

### `Ember.isNamespace`

No replacement, not needed.

### `Ember.toString`

No replacement, not needed.
e
### `Ember.Container`

Both value and type are not needed.

### `Ember.Registry` 

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

Use 
```js
import { setComponentManager } from '@ember/component';
```

### `Ember._componentManagerCapabilities`

Use
```js
import { capabilities } from '@ember/component';
```

### `Ember._modifierManagerCapabilities`

Use
```js
import { capabilities } from '@ember/modifier';
```

### `Ember.meta`

No replacement, not public API

### `Ember._createCache`

### `Ember._cacheGetValue`
### `Ember._cacheIsConst`
### `Ember._descriptor`
### `Ember._getPath`
### `Ember._setClassicDecorator`
### `Ember._tracked`
### `Ember.beginPropertyChanges`
### `Ember.changeProperties`
### `Ember.endPropertyChanges`
### `Ember.hasListeners`
### `Ember.libraries`
### `Ember._ContainerProxyMixin`
### `Ember._ProxyMixin`
### `Ember._RegistryProxyMixin`
### `Ember.ActionHandler`
### `Ember.Comparable`
### `Ember.RSVP`
### `Ember._Cache`
### `Ember.GUID_KEY`
### `Ember.canInvoke`
### `Ember.generateGuid`
### `Ember.guidFor`
### `Ember.uuid`
### `Ember.wrap`
### `Ember.getOwner`
### `Ember.onLoad`
### `Ember.runLoadHooks`
### `Ember.setOwner`
### `Ember.Application`
### `Ember.ApplicationInstance`
### `Ember.Namespace`
### `Ember.A`
### `Ember.Array`
### `Ember.NativeArray`
### `Ember.isArray`
### `Ember.makeArray`
### `Ember.MutableArray`
### `Ember.ArrayProxy`
### `Ember.FEATURES`
### `Ember._Input`
### `Ember.Component`
### `Ember.Helper`
### `Ember.Controller`
### `Ember.ControllerMixin`
### `Ember._captureRenderTree`
### `Ember.assert`
### `Ember.warn`
### `Ember.debug`
### `Ember.deprecate`
### `Ember.deprecateFunc`
### `Ember.runInDebug`
### `Ember.inspect`
### `Ember.Debug`
### `Ember.ContainerDebugAdapter`
### `Ember.DataAdapter`
### `Ember._assertDestroyablesDestroyed`
### `Ember._associateDestroyableChild`
### `Ember._enableDestroyableTracking`
### `Ember._isDestroying`
### `Ember._isDestroyed`
### `Ember._registerDestructor`
### `Ember._unregisterDestructor`
### `Ember.destroy`
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
