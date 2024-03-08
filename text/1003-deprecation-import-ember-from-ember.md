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

**If anyone one dependency in an app's dependency tree does `import ... from 'ember'`, every feature of the framework is shipped to your users, without any ability for you to optimize.**

By removing this set of exports, we have an opportunity to shrink some apps (as some APIs are not used), improving the load performance of ember apps -- and we removing all of these gives us a chance to have a better grasp of what we can get rid of permananently.

Many of these APIs already have alternatives, and those will be called out explicitly in the _Transition Path_ below.

## Transition Path

This list is semi-exhaustive, in that it covers _every_ export from 'ember', but may not exhaustivily provide alternatives.

Throughout the rest of this RFC, the following key will be used:
- 🌐 to mean "this is public API"
- 🔒 to mean "this is private API"
- 🧷 to mean "this is protected API"
- 🫣 to mean "no declared access"

### Testing utilities

APIs for wiring up a test framework (e.g. QUnit, _etc_)

|   | API | Usage: EmberObserver | -- | 
| - | --- | ----- | -- |
|🌐 | `Ember.Test` | [A good few](https://emberobserver.com/code-search?codeQuery=Ember.Test&sort=updated&sortAscending=false) | |
|🌐 | `Ember.Test.Adapter` | [`ember-cli-fastboot` and `@ember/test-helpers`](https://emberobserver.com/code-search?codeQuery=Ember.Test.Adapter&sort=updated&sortAscending=false) | currently available at [`@ember/test`](https://api.emberjs.com/ember/5.6/modules/@ember%2Ftest) |
|🌐 | `Ember.Test.QUnitAdapter` | [`ember-cli-fastboot`](https://emberobserver.com/code-search?codeQuery=Ember.Test.QUnitAdapter&sort=updated&sortAscending=false) | |
|🌐 | `Ember.setupForTesting` | [`ember-cli-fastboot`](https://emberobserver.com/code-search?codeQuery=Ember.setupForTesting&sort=updated&sortAscending=false) | | 

These will need to be moved to a module such as `@ember/test`.

### A way to communicate with the ember-inspector

The inspector will be hit especially hard by the removal of these APIs.

A good few already have available imports though.

|   | API | import |
| - | --- | ------ |
|🔒| `Ember.meta` | `import { meta } from '@ember/-internals/meta';` |
|🌐| `Ember.VERSION` | none, we should add one, `@ember/version` |
|🔒| `Ember._captureRenderTree` | `import { captureRenderTree } from '@ember/debug';` |
|🔒| `Ember.instrument` | `import { instrument } from '@ember/instrumentation';` |
|🔒| `Ember.subscribe` | `import { subscribe } from '@ember/instrumentation';` |
|🔒| `Ember.Instrumentation.*` | `import { * } from '@ember/instrumentation';`  |
|🫣| `Ember.ViewUtils` | `import * as viewUtils from '@ember/-internals/views';`[^view-utils]  |
|🔒| `Ember.ViewUtils.getChildViews` | `import { getChildViews } from '@ember/-internals/views';` |
|🫣| `Ember.ViewUtils.getElementView` | `import { getElementView } from '@ember/-internals/views';` |
|🔒| `Ember.ViewUtils.getRootViews` | `import { getRootViews } from '@ember/-internals/views';` |
|🔒| `Ember.ViewUtils.getViewBounds` | `import { getViewBounds } from '@ember/-internals/views';` |
|🔒| `Ember.ViewUtils.getViewBoundingClientRect` | `import { getViewBoundingClientRect } from '@ember/-internals/views';` |
|🔒| `Ember.ViewUtils.getViewClientRects` | `import { getViewClientRects } from '@ember/-internals/views';` |
|🔒| `Ember.ViewUtils.getViewElement`  | `import { getViewElement } from '@ember/-internals/views';` |
|🫣| `Ember.ViewUtils.isSimpleClick` | `import { isSimpleClick } from '@ember/-internals/views';` |
|🫣| `Ember.ViewUtils.isSerializationFirstNode` | `import { isSerializationFirstNode } from '@ember/-internals/glimmer';` |

[^view-utils]: Not all of these exports are used for `ViewUtils`.


Perhaps we can have folks add this to their apps:
```js
import { macroCondition, isDevelopingApp, importSync } from '@embroider/macros';

if (macroCondition(isDevelopingApp())) {
  // maybe this is side-effecting and installs 
  // some functions on `globalThis` that the inspector could call
  // since the inspector can't import modules from a built app.
  importSync('@ember/inspector-support');
}
```

### No replacements.

Applies to both the value and type exports (if applicable). All of these will not be re-exported from other `@ember/*` packages, but the following tables will show addon usage[^why-addon-usage] in the ecosystem and potential paths forward for library authors.

[^why-addon-usage]: Addons are notorious for doing things they shouldn't, accessing private APIs, doing crazy things so users don't have to, working around ecosystem and broader ecosystem problems etc. It's also expected that addon authors will be able to handle migrations more quickly than app devs.


|   | API | Usage: EmberObserver | Migration |
| - | --- | ----- | --------- |
|🫣 | `Ember._getPath` | [None](https://emberobserver.com/code-search?codeQuery=Ember._getPath) | n/a |
|🫣 | `Ember.isNamespace` | [None](https://emberobserver.com/code-search?codeQuery=Ember.isNamespace) | n/a |
|🫣 | `Ember.toString` | [None](https://emberobserver.com/code-search?codeQuery=Ember.toString) | n/a |
|🔒 | `Ember.Container` | [Many, but old or docs](https://emberobserver.com/code-search?codeQuery=Ember.Container) | n/a |
|🔒 | `Ember.Registry` | [Many, but old or docs](https://emberobserver.com/code-search?codeQuery=Ember.Registry) | n/a |

Internal decorator utils
|   | API | Usage: EmberObserver | Migration |
| - | --- | ----- | --------- |
|🫣 | `Ember._descriptor` | EmberObserver: [None](https://emberobserver.com/code-search?codeQuery=Ember._descriptor) | n/a |
|🔒 | `Ember._setClassicDecorator` | EmberObserver: [ember-concurrency](https://emberobserver.com/code-search?codeQuery=Ember._setClassicDecorator) | n/a |

Reactivity
|   | API | Usage: EmberObserver | Migration |
| - | --- | ----- | --------- |
|🔒 | `Ember.beginPropertyChanges` | [ember-m3 + old addons](https://emberobserver.com/code-search?codeQuery=Ember.beginPropertyChanges) | n/a |
|🔒 | `Ember.endPropertyChanges` | [ember-m3 + old addons](https://emberobserver.com/code-search?codeQuery=Ember.endPropertyChanges) | n/a |
|🔒 | `Ember.changeProperties` | [None](https://emberobserver.com/code-search?codeQuery=Ember.changeProperties) | n/a |

Observable 
|   | API | Usage: EmberObserver | Migration |
| - | --- | ----- | --------- |
|🌐 | `Ember.hasListeners` | [None](https://emberobserver.com/code-search?codeQuery=Ember.hasListeners) | n/a |

Mixins
|   | API | Usage: EmberObserver | Migration |
| - | --- | ----- | --------- |
|🔒 | `Ember._ContainerProxyMixin` | [mostly old addons](https://emberobserver.com/code-search?codeQuery=Ember._ContainerProxyMixin&sort=updated&sortAscending=false). Includes `ember-decorators`, `ember-data-has-many-query`, `ember-graphql-adapter`, `ember-cli-fastboot` (in tests / test-support) | n/a |
|🔒 | `Ember._RegistryProxyMixin` | [mostly old addons](https://emberobserver.com/code-search?codeQuery=Ember._RegistryProxyMixin&sort=updated&sortAscending=false). Includes `ember-decorators`, `ember-data-has-many-query`, `ember-graphql-adapter`, `ember-cli-fastboot` (in tests / test-support) | n/a |
|🔒 | `Ember._ProxyMixin` | [`ember-bootstrap-components`, 8 years ago](https://emberobserver.com/code-search?codeQuery=Ember._ProxyMixin&sort=updated&sortAscending=false) | n/a |
|🔒 | `Ember.ActionHandler` | ['ember-error-tracker' + old addons](https://emberobserver.com/code-search?codeQuery=Ember.ActionHandler&sort=updated&sortAscending=false). Many usages include pre-modules Ember usage. | n/a |
|🔒 | `Ember.Comparable` | [ember-data-model-fragments](https://emberobserver.com/code-search?codeQuery=Ember.Comparable&sort=updated&sortAscending=false) | n/a |


Utility
|   | API | Usage: EmberObserver | Migration |
| - | --- | ----- | --------- |
|🫣 | `Ember.lookup` | [old addons, > 6 years](https://emberobserver.com/code-search?codeQuery=Ember.lookup&sort=updated&sortAscending=false) | Use `getOwner(...).lookup` from [`@ember/owner`](https://api.emberjs.com/ember/5.6/functions/@ember%2Fowner/getOwner) |
|🌐 | `Ember.libraries` | [Many usages, mostly ember-data and related](https://emberobserver.com/code-search?codeQuery=Ember.libraries&sort=updated&sortAscending=false) | This isn't a behavior that Ember needs to provide, nor should it be library authors' responsibilty to register themselves with a library listing system. App authors could choose to use any webpack or other build plugin that collections this information, such as [webpack-node-modules-list](https://github.com/ubilabs/webpack-node-modules-list) or [unplugin-info](https://github.com/yjl9903/unplugin-info). | 
|🫣 | `Ember._Cache` | [None](https://emberobserver.com/code-search?codeQuery=Ember._Cache&sort=updated&sortAscending=false) | n/a |
|🔒 | `Ember.GUID_KEY` | [`ember-data-save-relationships`, 6 years ago](https://emberobserver.com/code-search?codeQuery=Ember.GUID_KEY&sort=updated&sortAscending=false) | n/a |
| 🔒 | `Ember.canInvoke` | [@summit-electric-supply](https://emberobserver.com/code-search?codeQuery=Ember.canInvoke&sort=updated&sortAscending=false) | use [optional chaining](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining), e.g.: `this.foo?.method?.()` |
|🔒 | `Ember.generateGuid` | [`ember-flexberry + old addons](https://emberobserver.com/code-search?codeQuery=Ember.generateGuid&sort=updated&sortAscending=false) | Use [`guidFor`](https://api.emberjs.com/ember/5.6/functions/@ember%2Fobject%2Finternals/guidFor) or [`uuid`](https://www.npmjs.com/package/uuid) or the browser-native [`crypto.randomUUID()`](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/randomUUID) |

|🌐 | `Ember.uuid` | [3 recent addons](https://emberobserver.com/code-search?codeQuery=Ember.uuid&sort=updated&sortAscending=false) | Use [`guidFor`](https://api.emberjs.com/ember/5.6/functions/@ember%2Fobject%2Finternals/guidFor) or [`uuid`](https://www.npmjs.com/package/uuid) or the browser-native [`crypto.randomUUID()`](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/randomUUID) |
|🔒 | `Ember.wrap` | [None](https://emberobserver.com/code-search?codeQuery=Ember.wrap&sort=updated&sortAscending=false) | n/a |
|🔒 | `Ember.inspect` | [old addons](https://emberobserver.com/code-search?codeQuery=Ember.inspect&sort=updated&sortAscending=false) | n/a | 
|🫣 | `Ember.Debug` | [old addons](https://emberobserver.com/code-search?codeQuery=Ember.Debug&sort=updated&sortAscending=false) | use [`@ember/debug`](https://api.emberjs.com/ember/5.6/modules/@ember%2Fdebug) | 
|🫣 | `Ember.cacheFor` | [old addons](https://emberobserver.com/code-search?codeQuery=Ember.cacheFor&sort=updated&sortAscending=false) | potentially [`@glimmer/tracking/primitives/cache`](https://api.emberjs.com/ember/5.6/modules/@glimmer%2Ftracking%2Fprimitives%2Fcache) |
|🌐 | `Ember.ComputedProperty` | [aside from docs, old addons](https://emberobserver.com/code-search?codeQuery=Ember.ComputedProperty&sort=updated&sortAscending=false). Most recent usage is 3 years ago in `ember-cli-furnance-validation` | n/a |
|🫣 | `Ember.RouterDSL` | [old addons](https://emberobserver.com/code-search?codeQuery=Ember.RouterDSL&sort=updated&sortAscending=false) | n/a |
|🔒 | `Ember.controllerFor` | [None](https://emberobserver.com/code-search?codeQuery=Ember.controllerFor&sort=updated&sortAscending=false) | n/a |
|🔒 | `Ember.generateController` | [bitbird-core-ember-routing, 5 years ago](https://emberobserver.com/code-search?codeQuery=Ember.generateController&sort=updated&sortAscending=false) | n/a |
|🔒 | `Ember.generateControllerFactory` | [None](https://emberobserver.com/code-search?codeQuery=Ember.generateControllerFactory&sort=updated&sortAscending=false) | n/a |

|   | API | Usage: EmberObserver | Migration |
| - | --- | ----- | --------- |
|🌐 | `Ember.VERSION` | EmberObserver: [Not many usages](https://emberobserver.com/code-search?codeQuery=Ember.VERSION&sort=updated&sortAscending=false). | This has the ember version in it, but it could be converted to a virtual module to import from somewhere, such as `@ember/version` |
|🔒 | `Ember._Backburner` | EmberObserver: [None](https://emberobserver.com/code-search?codeQuery=Ember._Backburner&sort=updated&sortAscending=false) | n/a |
|🌐 | `Ember.inject` | EmberObserver: [Many, all using classic classes](https://emberobserver.com/code-search?codeQuery=Ember.inject&sort=updated&sortAscending=false). A lot of results are also classic-class docs. | Use [`@service`](https://api.emberjs.com/ember/5.6/functions/@ember%2Fservice/service) |

Any projects using these are already not safe for embroider and won't work with Vite
|   | API | Usage: EmberObserver | Migration |
| - | --- | ----- | --------- |
|🫣 | `Ember.__loader` | [159 addons. Some experimental. Most from `@ascua`](https://emberobserver.com/code-search?codeQuery=Ember.__loader&sort=updated&sortAscending=false) | n/a |
| 🫣 | `Ember.__loader.require` | [same as `Ember.__loader`](https://emberobserver.com/code-search?codeQuery=Ember.__loader.require&sort=updated&sortAscending=false) | n/a |
| 🫣 | `Ember.__loader.define` | [5 addons, ~2 recent](https://emberobserver.com/code-search?codeQuery=Ember.__loader.define&sort=updated&sortAscending=false). One is `ember-cli-fastboot` (tests, test-support). | n/a |
|🫣 | `Ember.__loader.registry` | [13 addons, ~5 recent](https://emberobserver.com/code-search?codeQuery=Ember.__loader.registry&sort=updated&sortAscending=false). One is `ember-cli-fastboot` (tests, test-support). | n/a |
|🔒 | `Ember.BOOTED` | [None](https://emberobserver.com/code-search?codeQuery=Ember.BOOTED&sort=updated&sortAscending=false) | n/a |


Replaced by [RFC #931][RFC-931].
For scenarios where folks would like to compile templates at runtime, see [RFC #931][RFC-931] or the code of [ember-repl](https://www.npmjs.com/package/ember-repl).
|   | API | Usage: EmberObserver | Migration |
| - | --- | ----- | --------- |
|🔒 | `Ember.TEMPLATES` | [`ember-resolver`](https://emberobserver.com/code-search?codeQuery=Ember.TEMPLATES&sort=updated&sortAscending=false) | n/a |
|🫣 | `Ember.HTMLBars` | [Lots of usage (encompasses the below APIs)](https://emberobserver.com/code-search?codeQuery=Ember.HTMLBars&sort=updated&sortAscending=false) | n/a |
|🫣 | `Ember.HTMLBars.DOMHelper` | [`ember-cli-fastboot`](https://emberobserver.com/code-search?codeQuery=Ember.HTMLBars.DOMHelper&sort=updated&sortAscending=false) uses `protocolForURL`, `parseHTML` | n/a |
|🫣 | `Ember.HTMLBars.template` | [`ember-cli-fastboot`](https://emberobserver.com/code-search?codeQuery=Ember.HTMLBars.template&sort=updated&sortAscending=false) (and `ember-ast-hot-load`) | n/a |
|🫣 | `Ember.HTMLBars.compile` | [old addons](https://emberobserver.com/code-search?codeQuery=Ember.HTMLBars.compile&sort=updated&sortAscending=false) | n/a |
|🫣 | `Ember.HTMLBars.precomple` | [`ember-ast-hot-load`](https://emberobserver.com/code-search?codeQuery=Ember.HTMLBars.precompile&sort=updated&sortAscending=false) | n/a |
|🫣 | `Ember.Handlebars` | [174 addons, mostly @ascua](https://emberobserver.com/code-search?codeQuery=Ember.Handlebars&sort=updated&sortAscending=false) and also a lot of mentions in docs. | n/a |
|🫣 | `Ember.Handlebars.template` | [None](https://emberobserver.com/code-search?codeQuery=Ember.Handlebars.template&sort=updated&sortAscending=false) | n/a |
|🫣 | `Ember.Handlebars.Utils.escapeExpression` | [100 addons, mostly @ascua](https://emberobserver.com/code-search?codeQuery=Ember.Handlebars.Utils.escapeExpression&sort=updated&sortAscending=false) | Removed in [ember.js PR#20360](https://github.com/emberjs/ember.js/pull/20360) as it is not public API. |
|🫣 | `Ember.Handlebars.compile` | [`ember-cli-fastboot` and `ember-collection`](https://emberobserver.com/code-search?codeQuery=Ember.Handlebars.compile&sort=updated&sortAscending=false) | n/a |
|🫣 | `Ember.Handlebars.precomple` | [None](https://emberobserver.com/code-search?codeQuery=Ember.Handlebars.precompile&sort=updated&sortAscending=false) | n/a |


Other APIs
- 🫣 `Ember.testing`  
  Instead, use

  ```js
  import { macroCondition, isTesting } from '@embroider/macros';

  // ...

  if (macroCondition(isTesting())) {
    // test only code here
  }
  ```

- 🌐 `Ember.onerror`
  Instead use an event listener for the `error` event on window.
  ```js
  window.addEventListener('error', /* ... event handler ... */);
  ```


[RFC-931]: https://github.com/emberjs/rfcs/pull/931



#### Imports Available

Most of this is covered in [RFC #176](https://rfcs.emberjs.com/id/0176-javascript-module-api)

Unless otherwise state, there will not be usage-based decision on these, as they all exist under available imports today.

|   | `Ember.` API | Use this instead | 
| - | ------------ | ---------------- |
|🌐 | `Ember.FEATURES` | `import { isEnabled, FEATURES } from '@ember/canary-features';` | 
|🌐 | `Ember._setComponentManager` | `import { setComponentManager } from '@ember/component';` | 
|🌐 | `Ember._componentManagerCapabilities` | `import { capabilities } from '@ember/component';` | 
|🌐 | `Ember._modifierManagerCapabilities` | `import { capabilities } from '@ember/modifier';` | 
|🌐 | `Ember._createCache` | `import { createCache } from '@glimmer/tracking/primitives/cache';` [RFC #615][RFC-615] | 
|🌐 | `Ember._cacheGetValue` | `import { getValue } from '@glimmer/tracking/primitives/cache';` [RFC #615][RFC-615] | 
|🌐 | `Ember._cacheIsConst` | `import { isConst } from '@glimmer/tracking/primitives/cache';` [RFC #615][RFC-615] | 
|🌐 | `Ember._tracked` | `import { tracked } from '@glimmer/tracking';` | 
|🌐 | `Ember.RSVP` | `import RSVP from 'rsvp';` | 
|🌐 | `Ember.guidFor` | `import { guidFor } from '@ember/object/internals';` | 
|🌐 | `Ember.getOwner` | `import { getOwner } from '@ember/owner';` | 
|🌐 | `Ember.setOwner` | `import { setOwner } from '@ember/owner';` |
|🌐 | `Ember.onLoad` | `import { onLoad } from '@ember/application';` | 
|🌐 | `Ember.runLoadHooks` | `import { runLoadHooks } from '@ember/application';` | 
|🌐 | `Ember.Application` | `import Application from '@ember/application';` | 
|🌐 | `Ember.ApplicationInstance` | `import ApplicationInstance from '@ember/application/instance';` | 
|🌐 | `Ember.Namespace` | `import Namespace from '@ember/application/namespace';` | 
|🌐 | `Ember.A` | `import { A }  from '@ember/array';` | 
|🌐 | `Ember.Array` | `import Array  from '@ember/array';` |
|🌐 | `Ember.NativeArray` | `import { NativeArray }  from '@ember/array';` |
|🌐 | `Ember.isArray` | `import { isArray }  from '@ember/array';` |
|🔒 | `Ember.makeArray` | `import { makeArray }  from '@ember/array';` |
|🌐 | `Ember.MutableArray` | `import MutableArray  from '@ember/array/mutable';` |
|🌐 | `Ember.ArrayProxy` | `import ArrayProxy  from '@ember/array/proxy';` |
|🌐 | `Ember._Input` | `import { Input }  from '@ember/component';` |
|🌐 | `Ember.Component` | `import Component  from '@ember/component';` |
|🌐 | `Ember.Helper` | `import Helper  from '@ember/component/helper';` |
|🌐 | `Ember.Controller` | `import Controller  from '@ember/controller';` |
|🔒 | `Ember.ControllerMixin` | `import { ControllerMixin } from '@ember/controller';` |
|🌐 | `Ember.assert` | `import { assert } from '@ember/debug';` |
|🌐 | `Ember.warn` | `import { warn } from '@ember/debug';` |
|🌐 | `Ember.debug` | `import { debug } from '@ember/debug';` |
|🌐 | `Ember.deprecate` | `import { deprecate } from '@ember/debug';` |
|🫣 | `Ember.deprecateFunc` | `import { deprecateFunc } from '@ember/debug';` |
|🌐 | `Ember.runInDebug` | `import { runInDebug } from '@ember/debug';` |
|🌐 | `Ember.Debug.registerDeprecationHandler` | `import { registerDeprecationHandler } from '@ember/debug';` |
|🌐 | `Ember.ContainerDebugAdapter` | `import ContainerDebugAdapter from '@ember/debug/container-debug-adapter';` |
|🌐 | `Ember.DataAdapter` | `import DataAdapter from '@ember/debug/data-adapter';` |
|🌐 | `Ember._assertDestroyablesDestroyed` | `import { assertDestroyablesDestroyed } from '@ember/destroyable';` | 
|🌐 | `Ember._associateDestroyableChild` | `import { associateDestroyableChild } from '@ember/destroyable';` | 
|🌐 | `Ember._enableDestroyableTracking` | `import { enableDestroyableTracking } from '@ember/destroyable';` |
|🌐 | `Ember._isDestroying` | `import { isDestroying } from '@ember/destroyable';` | 
|🌐 | `Ember._isDestroyed` | `import { isDestroyed } from '@ember/destroyable';` |
|🌐 | `Ember._registerDestructor` | `import { registerDestructor } from '@ember/destroyable';` |
|🌐 | `Ember._unregisterDestructor` | `import { unregisterDestructor } from '@ember/destroyable';` |
|🌐 | `Ember.destroy` | `import { destroy } from '@ember/destroyable';` |
|🌐 | `Ember.Engine` | `import Engine from '@ember/engine';` |
|🌐 | `Ember.EngineInstance` | `import Engine from '@ember/engine/instance';` |
|🔒 | `Ember.Enumerable` | `import Enumerable from '@ember/enumerable';` |
|🔒 | `Ember.MutableEnumerable` | `import MutableEnumerable from '@ember/enumerable/mutable';` |
|🌐 | `Ember.Object` | `import Object from '@ember/object';` |
|🌐 | `Ember._action` | `import { action } from '@ember/object';` |
|🌐 | `Ember.computed` | `import { computed } from '@ember/object';` |
|🌐 | `Ember.defineProperty` | `import { defineProperty } from '@ember/object';` |
|🌐 | `Ember.get` | `import { get } from '@ember/object';` |
|🌐 | `Ember.getProperties` | `import { getProperties } from '@ember/object';` |
|🌐 | `Ember.notifyPropertyChange` | `import { notifyPropertyChange } from '@ember/object';` |
|🌐 | `Ember.observer` | `import { observer } from '@ember/object';` |
|🌐 | `Ember.set` | `import { set } from '@ember/object';` |
|🌐 | `Ember.trySet` | `import { trySet } from '@ember/object';` |
|🌐 | `Ember.setProperties` | `import { setProperties } from '@ember/object';` |
|🌐 | `Ember._dependentKeyCompat` | `import { dependentKeyCompat } from '@ember/object/compat';` |
|🌐 | `Ember.expandProperties` | `import { expandProperties } from '@ember/object/computed';` |
|🌐 | `Ember.CoreObject` | `import EmberObject from '@ember/object';` |
|🌐 | `Ember.Evented` | `import Evented from '@ember/object/evented';` |
|🌐 | `Ember.on` | `import { on } from '@ember/object/evented';` |
|🌐 | `Ember.addListener` | `import { addListener } from '@ember/object/events';` |
|🌐 | `Ember.removeListener` | `import { removeListener } from '@ember/object/events';` |
|🌐 | `Ember.sendEvent` | `import { sendEvent } from '@ember/object/events';` |
|🌐 | `Ember.Mixin` | `import Mixin from '@ember/object/mixin';` |
|🔒 | `Ember.mixin` | `import { mixin } from '@ember/object/mixin';` |
|🌐 | `Ember.Observable` | `import Observable from '@ember/object/observable';` |
|🌐 |`Ember.addObserver` | `import { addObserver } from '@ember/object/observers';` |
|🌐 | `Ember.removeObserver` | `import { removeObserver } from '@ember/object/observers';` |
|🌐 | `Ember.PromiseProxyMixin` | `import EmberPromiseProxyMixin from '@ember/object/promise-proxy-mixin';` |
|🌐 | `Ember.ObjectProxy` | `import ObjectProxy from '@ember/object/proxy';` |
|🧷 | `Ember.HistoryLocation` | `import HistoryLocation from '@ember/routing/history-location';` |
|🧷 | `Ember.HashLocation` | `import HashLocation from '@ember/routing/hash-location';` |
|🧷 | `Ember.NoneLocation` | `import NoneLocation from '@ember/routing/none-location';` |
|🌐 | `Ember.Route` | `import Route from '@ember/routing/route';` |
|🌐 | `Ember.run` | `import { run } from '@ember/runloop';` |
|🌐 | `Ember.Service` | `import Service from '@ember/service';` |
|🌐 | `Ember.compare` | `import { compare } from '@ember/utils';` |
|🌐 | `Ember.isBlank` | `import { isBlank } from '@ember/utils';` |
|🌐 | `Ember.isEmpty` | `import { isEmpty } from '@ember/utils';` |
|🌐 | `Ember.isEqual` | `import { isEqual } from '@ember/utils';` |
|🌐 | `Ember.isPresent` | `import { isPresent } from '@ember/utils';` |
|🌐 | `Ember.typeOf` | `import { typeOf } from '@ember/utils';` |
|🌐 | `Ember._getComponentTemplate` | `import { getComponentTemplate } from '@ember/component';` | 
|🌐 | `Ember._setComponentTemplate` | `import { setComponentTemplate } from '@ember/component';` | 
|🌐 | `Ember._helperManagerCapabilities` | `import { capabilities } from '@ember/helper';` | 
|🌐 | `Ember._setHelperManager` | `import { setHelperManager } from '@ember/helper';` | 
|🌐 | `Ember._setModifierManager` | `import { setModifierManager } from '@ember/modifier';` | 
|🌐 | `Ember._templateOnlyComponent` | `import templateOnly from '@ember/component/template-only';` | 
|🌐 | `Ember._invokeHelper` | `import { invokeHelper } from '@ember/helper';` | 
|🌐 | `Ember._hash` | `import { hash } from '@ember/helper';` | 
|🌐 | `Ember._array` | `import { array } from '@ember/helper';` | 
|🌐 | `Ember._concat` | `import { concat } from '@ember/helper';` | 
|🌐 | `Ember._get` | `import { get } from '@ember/helper';` | 
|🌐 | `Ember._on` | `import { on } from '@ember/modifier';` | 
|🌐 | `Ember._fn` | `import { fn } from '@ember/helper';` | 
|🌐 | `Ember.ENV` | `import MyEnv from '<my-app>/config/environment';` |


[RFC-615]: https://rfcs.emberjs.com/id/0615-autotracking-memoization

## Implementation Plan 

These can happen in any order

- Add deprecations to each `Ember.*` access
- Add the [Testing utilities](#testing-utilities) to `@ember/test`
- Add an `@ember/version` package to `ember-source`
- Add re-exports of private APIs, `ComputedProperty`, and `_setClassicDecorator` 
    These will still be deprecated on `Ember.`, and will be deprecated themselves as we progress through deprecating Ember Classic.
- Update ember-inspector to use imports for the internals and instrumentation APIs
- Add `@ember/inspector-support` to `ember-source` to manage things like `LIBRARIES`. 
    ```js
    import { libraries } from '@ember/inspector-support';

    libraries.add('ember-data', '5.3.1');
    // and/or
    libraries.addAll(depInfoFromPlugin);
    ```
- Add deprecation guide entries for each API

## How We Teach This

The guides already use the modern imports where available.

There is a place that needs updating, around advanced debugging, where folks configure Backburner to be in debug mode.
- https://guides.emberjs.com/release/applications/run-loop/#toc_where-can-i-find-more-information
- https://guides.emberjs.com/release/configuring-ember/debugging/#toc_errors-within-emberrunlater-backburner
  - Access to backburner here isn't relevant though because it's accessed from the `run` import from `@ember/runloop`

When using embroider and `staticEmberSource: true`, the benefits of not having this file can be realized in apps (as long as the app and all consumed addons do not import from 'ember')

Available Codemods

- https://github.com/ember-codemods/ember-modules-codemod (from the work of RFC 176)

## Deprecation Guide

- Separate ids for each API so that folks don't have to scroll too far to get to their migration path (if a migration path exists).
- Mostly using the above tables, but without the `Usage: EmberObserver` column.

## Drawbacks

n/a, to be more module-friendly, we must get rid of the `'ember'` import. 

## Alternatives

n/a

## Unresolved questions

n/a

Do our instrumentation and internals sub-packages have any SemVer guarantees? Or are we allowed to "do what we need to" and not care about _public-facing_ SemVer?
