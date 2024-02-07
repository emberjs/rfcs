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

### New Module Needed

APIs for wiring up a test framework (e.g. QUnit, _etc_)
- `Ember.Test`
- `Ember.Test.Adapter`
- `Ember.Test.QUnitAdapter`
- `Ember.setupForTesting`

These will need to be moved to a module such as `@ember/testing`.

### A way to communicate with the ember-inspector

The inspector will be hit especially hard by the removal of these APIs.

- `Ember.meta`
- `Ember.VERSION`
- `Ember._captureRenderTree` 
- `Ember.instrument`
- `Ember.subscribe`
- `Ember.Instrumentation.instrument`
- `Ember.Instrumentation.subscribe`
- `Ember.Instrumentation.unsubscribe`
- `Ember.Instrumentation.reset`
- `Ember.ViewUtils`
- `Ember.ViewUtils.getChildViews`
- `Ember.ViewUtils.getElementView`
- `Ember.ViewUtils.getRootViews`
- `Ember.ViewUtils.getViewBounds`
- `Ember.ViewUtils.getViewBoundingClientRect`
- `Ember.ViewUtils.getViewClientRects`
- `Ember.ViewUtils.getViewElement`
- `Ember.ViewUtils.isSimpleClick`
- `Ember.ViewUtils.isSerializationFirstNode`


Perhaps we can have folks add this to their apps:
```js
import { macroCondition, isDevelopingApp, importSync } from '@embroider/macros';

if (macroCondition(isDevelopingApp())) {
  // maybe this is side-effecting and installs 
  // some functions on `globalThis` that the inspector could call
  importSync('@ember/inspector-support');
}
```

### No replacements.

Applies to both the value and type exports (if applicable).

- `Ember._getPath`
- `Ember.isNamespace`
- `Ember.toString`
- `Ember.Container`
- `Ember.Registry`

Internal decorator utils
- `Ember._descriptor`
- `Ember._setClassicDecorator`

Reactivity
- `Ember.beginPropertyChanges`
- `Ember.changeProperties`
- `Ember.endPropertyChanges`

Observable 
- `Ember.hasListeners`

Mixins
- `Ember._ContainerProxyMixin`
- `Ember._ProxyMixin`
- `Ember._RegistryProxyMixin`
- `Ember.ActionHandler`
- `Ember.Comparable`

Utility
- `Ember.libraries` - 
   App authors could choose to use any webpack or other build plugin that collections this information, such as [webpack-node-modules-list](https://github.com/ubilabs/webpack-node-modules-list). This additionally means that V1 libraries that pushed themselves into `Ember.libraries` no longer need to worry about interacting with this or any similar API. 
- `Ember._Cache`
- `Ember.GUID_KEY`
- `Ember.canInvoke`  
    Instead use [optional chaining](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining):
    ```js
    this.foo?.method?.();
    ```
- `Ember.testing`  
  Instead, use

  ```js
  import { macroCondition, isTesting } from '@embroider/macros';

  // ...

  if (macroCondition(isTesting()) {
    // test only code here
  }
  ```

- `Ember.onerror`
  Instead use an event listener for the `error` event on window.
  ```js
  window.addEventListener('error', /* ... event handler ... */);
  ```
- `Ember.generateGuid`
- `Ember.uuid`
- `Ember.wrap`
- `Ember.FEATURES`  
    But this is useful when working with feature flags.
    This information could live on a specially named `globalThis` property, the enabled features could be emitted as a virtual module to import.
- `Ember.ControllerMixin`
- `Ember.deprecateFunc`
- `Ember.inspect`
- `Ember.Debug`
  Replaced by some of `@ember/debug` exports.
- `Ember.cacheFor`
- `Ember.ComputedProperty`
- `Ember.RouterDSL`
- `Ember.controllerFor`
- `Ember.generateController`
- `Ember.generateControllerFactory`
- `Ember.VERSION`  
    This has the ember version in it, but it could be converted to a virtual module to import from somewhere.
- `Ember._Backburner`
- `Ember.inject`
- `Ember.__loader`
- `Ember.__loader.require`
- `Ember.__loader.define`
- `Ember.__loader.registry`
- `Ember.BOOTED`
- `Ember.TEMPLATES`

Replaced by [RFC #931][RFC-931]
- `Ember.HTMLBars`
- `Ember.HTMLBars.template`
- `Ember.HTMLBars.compile`
- `Ember.HTMLBars.precomple`
- `Ember.Handlebars`
- `Ember.Handlebars.template`
- `Ember.Handlebars.Utils.escapeExpression`
    Removed in [ember.js PR#20360](https://github.com/emberjs/ember.js/pull/20360) as it is not public API.
- `Ember.Handlebars.compile`
- `Ember.Handlebars.precomple`


[RFC-931]: https://github.com/emberjs/rfcs/pull/931


### `Ember.lookup`

#### Imports Available

Most of this is covered in [RFC #176](https://rfcs.emberjs.com/id/0176-javascript-module-api)

| `Ember.` API | Use this instead |
| ---------- | ---------------- |
| `Ember._setComponentManager` | `import { setComponentManager } from '@ember/component';` |
| `Ember._componentManagerCapabilities` | `import { capabilities } from '@ember/component';` |
| `Ember._modifierManagerCapabilities` | `import { capabilities } from '@ember/modifier';` |
| `Ember._createCache` | `import { createCache } from '@glimmer/tracking/primitives/cache';` [RFC #615][RFC-615] |
| `Ember._cacheGetValue` | `import { getValue } from '@glimmer/tracking/primitives/cache';` [RFC #615][RFC-615] |
| `Ember._cacheIsConst` | `import { isConst } from '@glimmer/tracking/primitives/cache';` [RFC #615][RFC-615] |
| `Ember._tracked` | `import { tracked } from '@glimmer/tracking';` |
| `Ember.RSVP` | `import RSVP from 'rsvp';` |
| `Ember.guidFor` | `import { guidFor } from '@ember/object/internals';` |
| `Ember.getOwner` | `import { getOwner } from '@ember/owner';` |
| `Ember.setOwner` | `import { setOwner } from '@ember/owner';` |
| `Ember.onLoad` | `import { onLoad } from '@ember/application';` |
| `Ember.runLoadHooks` | `import { runLoadHooks } from '@ember/application';` |
| `Ember.Application` | `import Application from '@ember/application';` |
| `Ember.ApplicationInstance` | `import ApplicationInstance from '@ember/application/instance';` |
| `Ember.Namespace` | `import Namespace from '@ember/application/namespace';` |
| `Ember.A` | `import { A }  from '@ember/array';` |
| `Ember.Array` | `import Array  from '@ember/array';` |
| `Ember.NativeArray` | `import { NativeArray }  from '@ember/array';` |
| `Ember.isArray` | `import { isArray }  from '@ember/array';` |
| `Ember.makeArray` | `import { makeArray }  from '@ember/array';` |
| `Ember.MutableArray` | `import MutableArray  from '@ember/array/mutable';` |
| `Ember.ArrayProxy` | `import ArrayProxy  from '@ember/array/proxy';` |
| `Ember._Input` | `import { Input }  from '@ember/component';` |
| `Ember.Component` | `import Component  from '@ember/component';` |
| `Ember.Helper` | `import Helper  from '@ember/component/helper';` |
| `Ember.Controller` | `import Controller  from '@ember/controller';` |
| `Ember.assert` | `import { assert } from '@ember/debug';` |
| `Ember.warn` | `import { warn } from '@ember/debug';` |
| `Ember.debug` | `import { debug } from '@ember/debug';` |
| `Ember.deprecate` | `import { deprecate } from '@ember/debug';` |
| `Ember.runInDebug` | `import { runInDebug } from '@ember/debug';` |
| `Ember.Debug.registerDeprecationHandler` | `import { registerDeprecationHandler } from '@ember/debug';` |
| `Ember.ContainerDebugAdapter` | `import ContainerDebugAdapter from '@ember/debug/container-debug-adapter';` |
| `Ember.DataAdapter` | `import DataAdapter from '@ember/debug/data-adapter';` |
| `Ember._assertDestroyablesDestroyed` | `import { assertDestroyablesDestroyed } from '@ember/destroyable';` | 
| `Ember._associateDestroyableChild` | `import { associateDestroyableChild } from '@ember/destroyable';` | 
| `Ember._enableDestroyableTracking` | `import { enableDestroyableTracking } from '@ember/destroyable';` |
| `Ember._isDestroying` | `import { isDestroying } from '@ember/destroyable';` | 
| `Ember._isDestroyed` | `import { isDestroyed } from '@ember/destroyable';` |
| `Ember._registerDestructor` | `import { registerDestructor } from '@ember/destroyable';` |
| `Ember._unregisterDestructor` | `import { unregisterDestructor } from '@ember/destroyable';` |
| `Ember.destroy` | `import { destroy } from '@ember/destroyable';` |
| `Ember.Engine` | `import Engine from '@ember/engine';` |
| `Ember.EngineInstance` | `import Engine from '@ember/engine/instance';` |
| `Ember.Enumerable` | `import Enumerable from '@ember/enumerable';` |
| `Ember.MutableEnumerable` | `import MutableEnumerable from '@ember/enumerable/mutable';` |
| `Ember.Object` | `import Object from '@ember/object';` |
| `Ember._action` | `import { action } from '@ember/object';` |
| `Ember.computed` | `import { computed } from '@ember/object';` |
| `Ember.defineProperty` | `import { defineProperty } from '@ember/object';` |
| `Ember.get` | `import { get } from '@ember/object';` |
| `Ember.getProperties` | `import { getProperties } from '@ember/object';` |
| `Ember.notifyPropertyChange` | `import { notifyPropertyChange } from '@ember/object';` |
| `Ember.observer` | `import { observer } from '@ember/object';` |
| `Ember.set` | `import { set } from '@ember/object';` |
| `Ember.trySet` | `import { trySet } from '@ember/object';` |
| `Ember.setProperties` | `import { setProperties } from '@ember/object';` |
| `Ember._dependentKeyCompat` | `import { dependentKeyCompat } from '@ember/object/compat';` |
| `Ember.expandProperties` | `import { expandProperties } from '@ember/object/computed';` |
| `Ember.CoreObject` | `import EmberObject from '@ember/object';` |
| `Ember.Evented` | `import Evented from '@ember/object/evented';` |
| `Ember.on` | `import { on } from '@ember/object/evented';` |
| `Ember.addListener` | `import { addListener } from '@ember/object/events';` |
| `Ember.removeListener` | `import { removeListener } from '@ember/object/events';` |
| `Ember.sendEvent` | `import { sendEvent } from '@ember/object/events';` |
| `Ember.Mixin` | `import Mixin from '@ember/object/mixin';` |
| `Ember.mixin` | `import { mixin } from '@ember/object/mixin';` |
| `Ember.Observable` | `import Observable from '@ember/object/observable';` |
| `Ember.addObserver` | `import { addObserver } from '@ember/object/observers';` |
| `Ember.removeObserver` | `import { removeObserver } from '@ember/object/observers';` |
| `Ember.PromiseProxyMixin` | `import EmberPromiseProxyMixin from '@ember/object/promise-proxy-mixin';` |
| `Ember.ObjectProxy` | `import ObjectProxy from '@ember/object/proxy';` |
| `Ember.HistoryLocation` | `import HistoryLocation from '@ember/routing/history-location';` |
| `Ember.HashLocation` | `import HashLocation from '@ember/routing/hash-location';` |
| `Ember.NoneLocation` | `import NoneLocation from '@ember/routing/none-location';` |
| `Ember.Route` | `import Route from '@ember/routing/route';` |
| `Ember.run` | `import { run } from '@ember/runloop';` |
| `Ember.Service` | `import Service from '@ember/service';` |
| `Ember.compare` | `import { compare } from '@ember/utils';` |
| `Ember.isBlank` | `import { isBlank } from '@ember/utils';` |
| `Ember.isEmpty` | `import { isEmpty } from '@ember/utils';` |
| `Ember.isEqual` | `import { isEqual } from '@ember/utils';` |
| `Ember.isPresent` | `import { isPresent } from '@ember/utils';` |
| `Ember.typeOf` | `import { typeOf } from '@ember/utils';` |
| `Ember._getComponentTemplate` | `import { getComponentTemplate } from '@ember/component';` | 
| `Ember._setComponentTemplate` | `import { setComponentTemplate } from '@ember/component';` | 
| `Ember._helperManagerCapabilities` | `import { capabilities } from '@ember/helper';` | 
| `Ember._setHelperManager` | `import { setHelperManager } from '@ember/helper';` | 
| `Ember._setModifierManager` | `import { setModifierManager } from '@ember/modifier';` | 
| `Ember._templateOnlyComponent` | `import templateOnly from '@ember/component/template-only';` | 
| `Ember._invokeHelper` | `import { invokeHelper } from '@ember/helper';` | 
| `Ember._hash` | `import { hash } from '@ember/helper';` | 
| `Ember._array` | `import { array } from '@ember/helper';` | 
| `Ember._concat` | `import { concat } from '@ember/helper';` | 
| `Ember._get` | `import { get } from '@ember/helper';` | 
| `Ember._on` | `import { on } from '@ember/modifier';` | 
| `Ember._fn` | `import { fn } from '@ember/helper';` | 
| `Ember.ENV` | `import MyEnv from '<my-app>/config/environment';` |


[RFC-615]: https://rfcs.emberjs.com/id/0615-autotracking-memoization

## How We Teach This

The guides already use the modern imports where available.

There is a place that needs updating, around advanced debugging, where folks configure Backburner to be in debug mode.
- https://guides.emberjs.com/release/applications/run-loop/#toc_where-can-i-find-more-information
- https://guides.emberjs.com/release/configuring-ember/debugging/#toc_errors-within-emberrunlater-backburner
  - Access to backburner here isn't relevant though because it's accessed from the `run` import from `@ember/runloop`

When using embroider and `staticEmberSource: true`, the benefits of not having this file can be realized in apps (as long as the app and all consumed addons do not import from 'ember')

Available Codemods

- https://github.com/ember-codemods/ember-modules-codemod (from the work of RFC 176)

## Drawbacks

n/a, to be more module-friendly, we must get rid of the `'ember'` import. 

## Alternatives

n/a

## Unresolved questions

n/a