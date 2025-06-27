---
stage: accepted
start-date: 2025-06-27T00:00:00.000Z # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - framework
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
suite: 
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
suite: Leave as is
-->

<!-- Replace "RFC title" with the title of your RFC -->

# Ember API to enable Vite support in Ember Inspector

## Summary

Define the API Ember should expose to the Ember Inspector (and similar tools) so it's compatible with Vite.

## Motivation

The Ember Inspector is a browser extension that extends the capacity of regular debuggers for Ember specifically. It allows developers to inspect their Ember apps and view information like the version of Ember and Ember Data running, the components render tree, the data loaded on the page, the state of the different Ember object instances like services, controllers, routes... It's a practical and popular extension, widely used in the Ember community. Unfortunately, it’s incompatible with modern Ember apps building with Vite.

In Ember 6.7, Vite should become the default experience when generating a new Ember app. To keep the developer experience as qualitative as it is now, it's very important to get the Ember Inspector working. To achieve this, we need to rethink the way `ember.js` exposes the modules used by the Inspector: this is the purpose of this RFC.

## Current architecture

The purpose of the Inspector is to display information about the Ember app running on the page. To do so, it needs to retrieve this information somehow. The architecture involves both the Inspector itself and the inspected Ember app that depends on a version of ember-source:

![A picture of the architecture described in the following paragraph](/images/0000-ember-api-for-inspector.png)

The Inspector (on the right) is composed of two main pieces:

- The UI is an Ember app that displays the content of the Inspector window when it runs.
- The folder `ember_debug` is built into a script `ember_debug.js`. The Inspector injects this script into the page to connect to the inspected Ember app.

The incompatibility with Vite apps lies in how `ember_debug.js` (on the left) uses ember-source. For a long time, `ember-cli` expressed all the modules using AMD (Asynchronous Module Definition) and `requirejs` `define()` statements. Addons and applications could rely on AMD loading to use these modules. This is what the Inspector does. When using `@embroider/vite` to build the Ember app with Vite, ember-source is loaded as ESM (ECMAScript modules), and there's essentially no `requirejs` module support: the Inspector was designed to work with the AMD approach and breaks when we move to ESM.

In a nutshell, supporting Vite means fixing the bridge between ember-source and `ember_debug.js`.

## Proposed design

### Overview

To fix the interaction between ember-source and ember-inspector, we want to implement changes in both repositories:

- In **ember.js**: we want to implement an API to expose ESM modules from Ember. The exposed moules are those `ember_debug` needs to send relevant information to the Inspector UI. To do so, we want to introduce a new module `@ember/debug/inspector-support.js` that one can include in their Ember app. This module would define a global (e.g. `emberInspectorLoader`) which provides a function that loads the ESM modules.

- In **ember-inspector**: we want to implement the ability for `ember_debug` to import all modules from Ember as ESM modules. This should be done without breaking the previous AMD implementation because the Inspector should keep its current ability to inspect Classic apps built with Ember CLI and Broccoli. To do so, we want to use top-level `await` in a conditional block that would be executed when the AMD modules don't exist. Using top-level `await` implies to emit the `ember_debug` script itself as ESM (This constraint has already been partially answered by a recent refactor of the `ember_debug` build, which now relies on Rollup).

### Implementation

On the **ember.js** side, a new module would expose a global provinding a `load` function:

```js
globalThis.emberInspectorLoader = {
  async load() {
    return {
      Application: await import('@ember/application'),
      // Other modules...
    }
  }
}
```

Note that having this global variable existing in your app will simply define the `load()` function. Modules loading happen only when the function is executed, and this will happen on the Inspector side (see below). In other words, if it turns out the Inspector requires a few modules that were not yet involed in running the Ember app on the page, they will be loaded only when the developer starts the inspector.

On the **ember-inspector** side, the file that handles the interactions with ember.js would looks like this:

```js
// ember_debug/utils/ember.js

// Current code relying on AMD to load modules...

/* New conditional block
 * Among all the Ember modules, captureRenderTree is critical for the inspector to work.
 * If still not defined at this point, we can safely assume there's no AMD support. */
if (!captureRenderTree) {
  const internalEmberModules = await globalThis.emberInspectorLoader.load();
  Application = internalEmberModules.Application;
  // Other modules...
}

// Same exports as before
export {
  Application
  // Other modules...
}

```

- It has a new conditional block to fallback to ESM support.
- It calls the `globalThis.emberInspectorLoader.load()`, that we made available in the ember.js part.

(Note that the approach has been implemented in a proof of concept so we can make sure the Inspector works fine this way. PRs [ember.js#20892](https://github.com/emberjs/ember.js/pull/20892/files) and [ember-inspector#2625](https://github.com/emberjs/ember-inspector/pull/2625/files) allow testing a first iteration of the Vite support when used together.)

### Detailed API

The detailed API described below reflect what the Ember Inspector uses.

<details>

<summary>Expand to see the detailed API, with the full list of exports:</summary>

```js
globalThis.emberInspectorLoader = {
  async load() {
    return {
      Application: await import('@ember/application'),
      ApplicationNamespace: await import('@ember/application/namespace'),
      Array: await import('@ember/array'),
      ArrayMutable: await import('@ember/array/mutable'),
      ArrayProxy: await import('@ember/array/proxy'),
      Component: await import('@ember/component'),
      ComputedProperty: await import('@ember/object/computed'),
      Controller: await import('@ember/controller'),
      Debug: await import('@ember/debug'),
      EmberDestroyable: await import('@ember/destroyable'),
      EmberObject: await import('@ember/object'),
      EnumerableMutable: await import('@ember/enumerable/mutable'),
      InternalsEnvironment: await import('@ember/-internals/environment'),
      InternalsMeta: await import('@ember/-internals/meta'),
      InternalsMetal: await import('@ember/-internals/metal'),
      InternalsRuntime: await import('@ember/-internals/runtime'),
      InternalsUtils: await import('@ember/-internals/utils'),
      InternalsViews: await import('@ember/-internals/views'),
      Instrumentation: await import('@ember/instrumentation'),
      RSVP: await import('rsvp'),
      Runloop: await import('@ember/runloop'),
      ObjectInternals: await import('@ember/object/internals'),
      Service: await import('@ember/service'),
      ObjectCore: await import('@ember/object/core'),
      ObjectEvented: await import('@ember/object/evented'),
      ObjectProxy: await import('@ember/object/proxy'),
      ObjectObservable: await import('@ember/object/observable'),
      ObjectPromiseProxyMixin: await import('@ember/object/promise-proxy-mixin'),
      VERSION: await import('ember/version'),
      GlimmerComponent: await import('@glimmer/component'),
      GlimmerManager: await import('@glimmer/manager'),
      GlimmerReference: await import('@glimmer/reference'),
      GlimmerRuntime: await import('@glimmer/runtime'),
      GlimmerUtil: await import('@glimmer/util'),
      GlimmerValidator: await import('@glimmer/validator'),
    }
  }
}
```

</details>

### Explicit import in the Ember app & polyfill

Using the new `@ember/debug/inspector-support.js` module in a Vite app requires to explicitly import it:

```js
// my-ember-app/app/app.[js,ts]

import '@ember/debug/inspector-support';
```

This import will become available in the release that includes the implementation presented in the former section. However, Vite support for Ember apps is available back to 3.28. It means that people using Vite from 3.28 to whatever version includes this RFC will have a non-functional inspector. We intend to solve this issue using a polyfil implemented in a dedicated package. If an Ember app relies on a version of ember-source that doesn't include the new API, the import above should be changed to:

```js
// my-ember-app/app/app.[js,ts]

import '@embroider/inspector-support-polyfill';
```

The polyfill will be in charge of providing the script that contains the API. Additionnally:
- It will adapt the content to the ember-source version if necessary (e.g We identified a module path which is different in version 4.8 and lower)
- It will error if the ember-source version includes the new API, and teach developers they should remove the dependency to the polyfill and replace the import with `'@ember/debug/inspector-support`.

## Related concerns

### Mixins deprecation

Mixins are currently being deprecated (see #1111 to #1117). However, the detailed API above import mixins. This is because it would be preferable to implement the present RFC before mixins are actually removed. Removing the mixins is a breaking change that would likely be released in Ember 7. On the other hand, Vite could become the default way to build Ember apps from 6.7: having a functional Vite support ready by this time would preserve a good developer experience for people creating new Ember apps.

Since mixins currently exist, and since the Inspector relies on them to render the correct information, it appears legit to include the mixins in the API. This way, the inspector keeps working correctly in Ember 6 and lower. Therefore, removing the mixins from the API would be part of #1111 to #1117 implementation. With the mixins gone, the Inspector code may require adjustment to display correclty the new without-mixin objects, but this should be ready for the major version that will remove them.
