---
stage: accepted
start-date: 2025-06-27T00:00:00.000Z # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1119
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

In Ember 6.7, Vite should become the default experience when generating a new Ember app. To keep the developer experience as qualitative as it is now, it's very important to get the Ember Inspector working. To achieve this, we need to rethink how `ember.js` exposes the modules used by the Inspector: this is the purpose of this RFC.

## Current architecture

The purpose of the Inspector is to display information about the Ember app running on the page. To do so, it needs to retrieve this information somehow. The architecture involves both the Inspector itself and the inspected Ember app that depends on a version of ember-source:

![A picture of the architecture described in the following paragraph](/images/1119-ember-api-for-inspector.png)

The Inspector (on the right) is composed of two main pieces:

- The UI is an Ember app that displays the content of the Inspector window when it runs.
- The folder `ember_debug` is built into a script `ember_debug.js`. The Inspector injects this script into the page to connect to the inspected Ember app.

### Vite support issue

The incompatibility with Vite apps lies in how `ember_debug.js` (on the left) uses ember-source. For a long time, `ember-cli` expressed all the modules using AMD (Asynchronous Module Definition) and `requirejs` `define()` statements. Addons and applications could rely on AMD loading to use these modules. This is what the Inspector does. When using `@embroider/vite` to build the Ember app with Vite, ember-source is loaded as ESM (ECMAScript modules), and there's essentially no `requirejs` module support: the Inspector was designed to work with the AMD approach and breaks when we move to ESM.

In a nutshell, supporting Vite means fixing the bridge between ember-source and `ember_debug.js`.

### Global approach issue

One important underlying problem with the current architecture is that most of the logic in `ember_debug.js` is the consequence of Ember not providing an API for tools like the Inspector. There's no Ember function that returns _the description of the application_ as a POJO (or JSON or whatever), so the `ember_debug.js` script creates its own description using the AMD modules available, with downsides:

- Since Ember doesn't expose a public API, Ember doesn't have any legitimate responsibility about what modules are available or not. If Ember code is reorganized and a module used by the Inspector has its path changed, there's no signal to tell the Inspector will break: it's the Inspector's problem to be presented with a fait accompli and align with the latest released version of Ember.

- The Inspector's code contains complex conditional pieces to support different Ember versions as the framework internals evolve. What piece of code supports what version is not explicitly indicated and the code is hard to maintain overall. Also, it happens that latest changes in the framework are not reflected, and tests are not always able to catch the issues (e.g. in Ember Inspector 4.13.1, most services are marked as computed properties because the execution path that marks services is no longer taken).

### Conclusion

Fixing the interactions between ember-source and ember-inspector to support Vite requires changes in both ember.js and ember-inspector: the former must expose the modules the inspector need as ESM, the latter must be able to consume ESM modules. However, implementing a new API in ember-source require to think this API future-proof. And future-proof implies that not only we solve the Vite support issue, but also the global approach issue. This is the reason why there are two different aspect to this RFC:

- The long-term goal aspect: Describe the API to implement in ember.js. This API should describe objects a way that match the Inspector requirements.
 
- The compatibility aspect: Vite support for Ember apps is available from 3.28 to latest. It means that at the time of writing, people who migrated their app to build with Vite are stuck with a non-functional Inspector. We must fix the support for all of these versions, which don't and will never have the new API describing objects. In other words, we need a compat API to make the bridge between old Ember versions and Vite world.

## Long-term design

### Overview

// TODO

### Implementation

// TODO

### Explicit import in the Ember app

Using the new `@ember/debug/inspector-support` module in a Vite app requires explicitly importing it:

```js
// my-ember-app/app/app.[js,ts]

import '@ember/debug/inspector-support';
```

This import will become available in the release that includes the implementation presented in the former section. If an Ember app relies on a version of ember-source that doesn't include the new API, the import above should be changed to:

```js
// my-ember-app/app/app.[js,ts]

import '@embroider/compat/inspector-support';
```

The next section will detail the content of `@embroider/compat/inspector-support`.


## Compatibility design

⚠️ Whereas the "Long-term design" section aims at defining the best API possible for the future, the present "Compatibility" section is more time sensitive. As explained earlier, Vite is already there, it should become the default in Ember 6.7, and the Inspector is currently broken for Ember+Vite developers. That's the reason why this section proposes a solution that is easy to implement in a short delay without reinventing entirely the communication between ember-source and ember-inspector.

### Overview

To fix the interaction between ember-source and ember-inspector for versions that don't have the new API yet, we want to use the concept of "compat" functionnality introduced in Embroider. The package `@embroider/compat` is what makes the bridge between classic Ember and Vite world.

- In **@embroider/compat**: we want to implement a compat API to expose ESM modules from Ember. The exposed modules are those `ember_debug` - as currently implemented - needs to send relevant information to the Inspector UI. To do so, we want to introduce a new module `@embroider/compat/inspector-support` that one can include in their Ember app. This module would define a global (e.g. `emberInspectorLoader`) which provides a function that loads the ESM modules.

- In **ember-inspector**: we want to implement the ability for `ember_debug` to import all the modules from Ember as ESM modules. This should be done without breaking the previous AMD implementation because the Inspector should keep its current ability to inspect Classic apps built with Ember CLI and Broccoli. To do so, we want to use top-level `await` in a conditional block that would be executed when the AMD modules don't exist. Using top-level `await` implies emitting the `ember_debug` script itself as ESM (This constraint has already been partially answered by a recent refactor of the `ember_debug` build, which now relies on Rollup).

### Implementation

On the **@embroider/compat** side, a new module `@embroider/compat/inspector-support` would expose a global providing a `load` function:

```js
globalThis.emberInspectorLoader = {
  async load() {
    const [
      Application,
      // Other names...
    ] = await Promise.all([
      import('@ember/application'),
      // Other imports...
    ])
    return {
      Application,
      // Other modules...
    }
  }
}
```

Note that having `emberInspectorLoader` global variable in your app will simply define the `load()` function. Modules will load only when the function is executed, and this will occur on the Inspector side (see below). In other words, if it turns out the Inspector requires a few modules that were not yet involved in running the Ember app on the page, they will be loaded only when the developer starts the inspector.

On the **ember-inspector** side, the file that imports the ESM modules would look like this:

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
- It calls the `globalThis.emberInspectorLoader.load()` that we made available in the ember.js part.

(Note that the approach has been implemented in a proof of concept so we can make sure the Inspector works fine this way. PRs [ember.js#20892](https://github.com/emberjs/ember.js/pull/20892/files) and [ember-inspector#2625](https://github.com/emberjs/ember-inspector/pull/2625/files) allow testing a first iteration of the Vite support when used together.)

### Detailed compat API

The detailed API described below reflects what the Ember Inspector uses.

<details>

<summary>Expand to see the detailed API, with the full list of exports:</summary>

```js
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
```

</details>

### Ember versions concerns

Since `'@ember/debug/inspector-support` fits the way the Inspector currently works, the content should fit each supported Ember version depending on modules availability:

- The content of the script will adapt to the ember-source version relying on `@embroider/macros` (e.g. We identified a module path which is different in version 4.8 and lower).

- This philosophy also applies to changes that could be done in future versions. For instance Mixins are currently being deprecated (see [#1111 to #1117](https://github.com/emberjs/rfcs/pulls?q=is%3Apr+author%3Awagenet+created%3A%3E2025-06-01+)). Removing the mixins is a breaking change that would likely be released in Ember 7. However, the compat API above imports mixins because the Inspector currently use them. If we can't get the Long-term API ready for Ember 7, then removing the mixins from the compat API and defining a macro condition to have them in 6+ would be part of [#1111 to #1117](https://github.com/emberjs/rfcs/pulls?q=is%3Apr+author%3Awagenet+created%3A%3E2025-06-01+) implementation.

- An error will be thrown if the ember-source version includes the new API, and teach developers they should remove the dependency on the polyfill and replace the import with `'@ember/debug/inspector-support`.
