---
Stage: Accepted
Start Date: 2022-01-07
Release Date: Unreleased
Release Versions:
ember-source: vX.Y.Z
ember-data: vX.Y.Z
Relevant Team(s): Ember.js, Ember CLI
RFC PR: https://github.com/emberjs/rfcs/pull/784
---

# `no-globals` optional feature

## Summary

Due to the use of the global namespace, especially for its AMD-style module system, Ember apps have problems coexisting
with other Ember apps or any AMD- or CJS-based JavaScript within the same page/context. This RFC proposes a way to prevent 
all potentially conflicting usage of these globals.

## Motivation

The most common case for running an Ember app is to run it as a pure isolated SPA (Single Page Application), i.e. to have
it *own* the HTML page it runs on. This is what you get out of the box when running `ember build`: a (mostly empty)
index.html, that refers to the build assets that run the Ember app inside that empty page.

But another use case is to [embed it into an existing page](https://guides.emberjs.com/release/configuring-ember/embedding-applications/),
so that it runs alongside other JavaScript (let's call it "3rd party" from the perspective of the Ember app). While that
*might* work fine in some cases, due to Ember's still present reliance on globals, especially its AMD-style module
system that is actually *incompatible* with the original one of [RequireJS](https://requirejs.org/), there is a high
chance of non-trivial conflicts.

Let's look at a few examples:

### Embedding an Ember app into a 3rd party HTML page

You might want to embed an Ember app on a 3rd party page (i.e. one that you don't control), for example
* a messaging/chat widget embedded on a corporate/sales website (think of Intercom)
* a product configurator embedded on an e-commerce site (that's what we do)
* a messaging board app on a customer support website (think of Discourse, but embedded)

In these cases, if the 3rd party page uses RequireJS itself, only one implementation (Ember vs. RequireJS) of the globally
defined `define()` and `require()` functions will "win", and the other's module loading will inherently be broken, in
the worst case both.

This is not uncommon, for example [Magento](https://magento.com/), the most wide-spread e-commerce solution worldwide,
[uses RequireJS](https://devdocs.magento.com/guides/v2.4/javascript-dev-guide/javascript/requirejs.html), so effectively
you cannot use an Ember app on its pages without *somehow* tackling that conflict.

### Micro Frontends

A rather new pattern in frontend development is the analogy of microservices for the frontend world, coined
[Micro Frontends](https://martinfowler.com/articles/micro-frontends.html), where you develop, test and deploy parts of
your frontend independently, and compose them together for a seamless user experience.

The same issue arises when one of those micro frontends happens to use RequireJS, or two or more frontends are actual
Ember apps. While in the latter case we don't have to deal with different implementations of `define()` and `require()`,
those functions close over their module registry, so whatever functions win (as being the globally defined one), they will
only be able to see *their* registered modules, the modules of the other app will not exist anymore. And we are not even
speaking about conflicting versions of the same modules (say `@ember/component`). So in any way, that scenario is bound
to fail miserably.

### Running an Ember app in a node.js environment

Node.js introduced its own module system, CommonJS (CJS), which happens to also define a global `require()` function.
When an Ember app runs in a node.js environment, it is able to `require()` and run node.js code. This obviously leads to
conflicts with Ember's AMD-style `require()`.

In practice this is a problem for [ember-electron](https://ember-electron.js.org/), which is able to package an Ember app
as a native Desktop app, and enable it to access (native) node.js modules. To work around the conflict, `ember-electron`:

* injects [a script](https://github.com/adopted-ember-addons/ember-electron/blob/464b13c76a89b9803cf27fcd974a798e05cb109c/public/shim-head.js) into `index.html` before the app and vendor scripts that renames `window.require` (the Node.js require) to `window.requireNode`
* injects [a script](https://github.com/adopted-ember-addons/ember-electron/blob/464b13c76a89b9803cf27fcd974a798e05cb109c/vendor/wrap-require.js) into `vendor.js` (after the Ember code) that replaces `window.require` (which is now the `loader.js` require) with a custom `require` function that tries requiring via both module loaders (Node.js and `loader.js`)

With this RFC implemented, there would still be some ambiguity where bundled app code calls `require()`, as that could refer
both to Ember's AMD version of it and to node's CJS. But using the already [recommended `requireNode()`](https://ember-electron.js.org/docs/guides/development-and-debugging#require-and-requirenode-) 
instead would resolve that ambiguity. The other case however, where 3rd party node modules expect the global `require()`
to be the CJS version and therefore forcing ember-electron into overloading Ember's global `require()` to also support node 
modules, would be solved by this RFC, as Ember wouldn't override node's global `require()` anymore.

## Detailed design

The guiding principle suggested in this RFC is to make Ember apps practically side effect free, with respect to the
usage of the browser's global namespace (`window`/`globalThis`).

That means it should be discouraged to
* read custom variables from the global namespace
* write anything to the global namespace

While this can only be a recommended but not enforceable best practice for user-land code, for Ember and its wider build
tooling ecosystem this should become a rule. That specifically includes:
* Ember.js (its runtime code)
* Ember-CLI
* ember-resolver
* ember-auto-import
* Embroider
* ember-engines

Where this is practically not possible, it should be ensured that there is no realistic chance of a conflict, by
* naming global variables in a unique way, e.g. scoped to the app's name (in `package.json`), so that two different Ember
  apps cannot cause conflicts
* ideally also adding a non-deterministic part like a random string that changes *per build*, that is only known to
  privileged code (as listed above), so user-land code cannot rely on the existence of these globals.

While the `Ember` global has already been deprecated and removed from Ember 4 as part of [RFC 706](https://emberjs.github.io/rfcs/0706-deprecate-ember-global.html),
there are still other variables that Ember apps leak into the global namespace as described below, most notably Ember's
AMD-like module system, and as such pose a risk for conflicts.

### Optional feature `no-globals`

The changes proposed here should ideally only require changes in Ember's build tooling. For idiomatic Ember code, especially
code that only uses ES modules, it is a non-breaking change. Also code that uses Ember's AMD-style module system
directly like `require()` or `require.entries` should continue to work without any changes in most cases, as long as the
code is built by Ember-CLI (see below).

> Although the AMD-like module API might not be strictly a public API of Ember, but it is often used in the wild to be at
> least regarded as "intimate API". So for the scope of this RFC, we will consider it equivalent to public API.

But there might be cases where people are doing things differently, and code - even after the build tooling changes
outlined below - still effectively relies on the globally defined AMD-like module API. For this reason we may not just
remove these from the global namespace, breaking backwards compatibility.

So this RFC proposes an opt-in behavior, by introducing a new "feature" based on the already well known
[`@ember/optional-features`](https://github.com/emberjs/ember-optional-features), namely `no-globals`.

All the following proposals only apply when a user has explicitly opted into the new behavior, by enabling that optional
feature!

> Note that similar to how Ember transitioned away from using jQuery, we might want to make this new behavior the default
> at some point, and deprecate the previous one. But this is out of scope for this RFC.


### Remove globals usage

As described above, when opting into `no-globals` mode, an Ember app built using Ember-CLI, including optionally
ember-auto-import, Embroider or ember-engines, should not read from or leak any globals, with the exceptions mentioned above.

The following identifies current global usage, and outlines possible ways to remove their usage, or at least prevent the
chance of conflicts. Note that this RFC does not aim to prescribe a specific implementation, as besides the visible
change in behavior of not relying on globals there is no public API introduced here, so the actual implementation remains
the task of any implementor of this RFC. The implementation outlines are merely intended to demonstrate the feasibility
of the proposed changes.

#### AMD-like module system

The global functions/namespaces `define` and `require` (and its aliases `requireModule`, `require`, `requirejs`) are set
as globals and provided by the [`loader.js`](https://github.com/ember-cli/loader.js) addon, which is part of the default
blueprint of any Ember app. Not having this addon or any equivalent replacement will break the app, as a.o. any `define()`
calls in `vendor.js` (for addon code) or `app.js` (for app code) would call an undefined function. So it is effectively a
mandatory dependency currently.

Despite that, Ember comes with its own (stripped down) [loader library](https://github.com/emberjs/ember.js/blob/master/packages/loader/lib/index.js),
that has been used (pre Ember 3.27) internally only. As of Ember 3.27 it is effectively unused when `loader.js` is present
(which it has to be, see above), making it dead code.

In `no-globals` mode, all of Ember's build tooling, besides EmberCLI also the optional but more or less "official"
extensions ember-auto-import, Embroider and ember-engines, should compile the JavaScript artifacts into a format that
does not rely on globally defined `define` and `require`. They should still be left unchanged as `loader.js` provides
them today, they should just not be set on the global namespace anymore.

> Note that this RFC does not propose a new packaging format for Ember apps that does not rely on AMD-like modules.
> While that could be a potential case for a future RFC, it is not in scope for this one. This only suggests to not rely
> on the module system being accessed as globals, or at least as described above not in a way that poses a risk for conflicts.

The following describes possible changes in the compilation format that should fulfil the above goals. Again, these are
implementation details, which are not prescribed in this RFC but hopefully useful for any future implementor, so feel free
to skip over this section:

---

<details>
<summary>Compilation changes</summary>

A typical build of `vendor.js` will look basically like this:

<details>
<summary>vendor.js</summary>

```js
// EmberENV code, ommited here, see following chapter

var loader, define, requireModule, require, requirejs;

(function(global) {
  
  // loader.js IIFE defing the globals above
  // see https://github.com/ember-cli/loader.js/blob/master/lib/loader/loader.js

})(this);

(function() {
  var define, require;
  
  (function () {
    // Ember's own internal loader code
  
    // ...
    // globalObj = globalThis w/ polyfill
    if (typeof globalObj.define === 'function' && typeof globalObj.require === 'function') {
      define = globalObj.define;
      require = globalObj.require;
  
      return;
    }
    
    // Following are Ember's own (internal) implementation of define and require
    // However due to the early return above and the fact that loader.js always prepends its code, thus assigns
    // the define and require globals, this code is never reached! (when the loader.js addon is present!)
    
  })();
  define("@ember/-internals/bootstrap/index", /*...*/);
  // ... 
  // all of Ember's own modules
  // ...
  require('@ember/-internals/bootstrap')
})();

define('@ember-data/adapter/-private', /* ... */);
// ... 
// all modules from Ember addons
// ...
```
</details>

And this is what a typical compilation of `app.js` will look like:

<details>
<summary>app.js</summary>

```js
define("my-app/app", /* ... */);
// ... 
// define() for all app modules
// ...
```
</details>

As you can see it relies on a globally set `define`, that has been declared in the first line of `vendor.js`, and [set
by the code of `loader.js`](https://github.com/ember-cli/loader.js/blob/ed8c85a713b0dac23bc974e0850863412304c1b1/lib/loader/loader.js#L46).

As to how to prevent the global usage while retaining compatibility of any code that uses those functions *as globals*,
we can make the build system emit the existing code wrapped in a [IIFE](https://developer.mozilla.org/en-US/docs/Glossary/IIFE),
that avoids polluting the global namespace, while still any calls of `define()` or `require()` will see the same functions,
they just do not come from the global namespace anymore.

This works for `vendor.js`, where these functions are defined, but not for `app.js` which needs access to the *same* functions
as defined in `vendor.js`. That's why we still need *some* usage of globals, but as outlined in the previous chapter, we
must make sure they are scoped to the current app, so they pose no risk of conflicts.

So for `vendor.js` this is what the basic change could look like:


<details>
<summary>vendor.js</summary>

```diff
+ (function(){
var loader, define, requireModule, require, requirejs;

(function(global) {
  
  // loader.js IIFE defing the globals above
  // see https://github.com/ember-cli/loader.js/blob/master/lib/loader/loader.js

})(this);
+ globalThis.__my-app_12345_define = define;
+ globalThis.__my-app_12345_require = require;
- (function() {
-   var define, require;
-   
-   (function () {
-     // Ember's own internal loader code
-   
-     // ...
-     // globalObj = globalThis w/ polyfill
-     if (typeof globalObj.define === 'function' && typeof globalObj.require === 'function') {
-       define = globalObj.define;
-       require = globalObj.require;
-   
-       return;
-     }
-     
-     // Following are Ember's own (internal) implementation of define and require
-     // However due to the early return above and the fact that loader.js always prepends its code, thus assigns
-     // the define and require globals, this code is never reached! (when the loader.js addon is present!)
-     
-   })();
  define("@ember/-internals/bootstrap/index", /*...*/);
  // ... 
  // all of Ember's own modules
  // ...
  require('@ember/-internals/bootstrap')
- })();

define('@ember-data/adapter/-private', /* ... */);
// ... 
// all modules from Ember addons
// ...
+ })();
```
</details>

> Note that we were able to drop all of Ember's internal module code here!

And this for `app.js`:

<details>
<summary>app.js</summary>

```diff
+ (function() {
+ var define = globalThis.__my-app_12345_define;
+ var require = globalThis.__my-app_12345_require;
define("my-app/app", /* ... */);
// ... 
// all app modules
// ...
+ })();
```
</details>

By wrapping the code in the IIFE and defining `define` and `require` within it, we are basically hiding any eventually defined
globals (e.g. 3rd party RequireJS), while letting all existing code use *our* implementation as if it were globally defined.

> Note that this does only work when code uses these functions as implicit globals. When explicitly referring to the global
> namespace like `window.require()` or `globalThis.require()`, then this will inevitably fail.

Besides these two bundles, which make up "classic" Ember apps, we must also take into account the additional bundles
("chunks") that ember-auto-import and Embroider would emit.

> As ember-auto-import and Embroider emit similar webpack-compiled chunks, we will only highlight the required changes
> in the case of ember-auto-import here.

ember-auto-import emits a `chunk.app.xxx.js` chunk, which includes the boundary between Ember's AMD-like modules and
whatever Webpack emits for the imported modules it handled, so this is basically the glue between these two worlds. It typically
looks something like this (cleaned up and simplified):

<details>
<summary>chunk.app.xxx.js</summary>

```js
var __ember_auto_import__;
(() => {
  var __webpack_modules__ = ({

    "../../../../private/var/folders/q3/yzggf_l965zbzps_bk8xr37c0000gn/T/broccoli-4171121iPrUeBlpJ8/cache-307-webpack_bundler_ember_auto_import_webpack/app.js":
      ((module, __unused_webpack_exports, __webpack_require__) => {
        module.exports = (function() {
          var d = _eai_d;
          var r = _eai_r;
          window.emberAutoImportDynamic = function(specifier) {
            if (arguments.length === 1) {
              return r("_eai_dyn_" + specifier);
            } else {
              return r("_eai_dynt_" + specifier)(Array.prototype.slice.call(arguments, 1));
            }
          };
          window.emberAutoImportSync = function(specifier) {return r("_eai_sync_" + specifier)(Array.prototype.slice.call(arguments, 1));};
          d("lodash-es/find", [], function() { return __webpack_require__(/*! lodash-es/find */ "./node_modules/lodash-es/find.js");});
          d("lodash-es/startsWith", [], function() {return __webpack_require__(/*! lodash-es/startsWith */ "./node_modules/lodash-es/startsWith.js");});
          d("_eai_dyn_lodash-es/concat", [], function() {return __webpack_require__.e(/*! import() */ "node_modules_lodash-es_concat_js").then(__webpack_require__.bind(__webpack_require__, /*! lodash-es/concat */ "./node_modules/lodash-es/concat.js"));});
        })();
      }),

    "../../../../private/var/folders/q3/yzggf_l965zbzps_bk8xr37c0000gn/T/broccoli-4171121iPrUeBlpJ8/cache-307-webpack_bundler_ember_auto_import_webpack/l.js":
      (function(module, exports) {
        window._eai_r = require;
        window._eai_d = define;
      })
  });
  // ... webpack internals 
})();
```
</details>

Note that ember-auto-import itself creates new globals `_eai_*` here, that we must prevent. And it takes Ember's `require`
and `define` from the global namespace. A "fixed" version could look like this:

<details>
<summary>chunk.app.xxx.js</summary>

```diff
var __ember_auto_import__;
(() => {
  var __webpack_modules__ = ({

    "../../../../private/var/folders/q3/yzggf_l965zbzps_bk8xr37c0000gn/T/broccoli-4171121iPrUeBlpJ8/cache-307-webpack_bundler_ember_auto_import_webpack/app.js":
      ((module, __unused_webpack_exports, __webpack_require__) => {
        module.exports = (function() {
-          var d = _eai_d;
+          var d = globalThis.__ember_12345_define;
-          var r = _eai_r;
+          var r = globalThis.__ember_12345_require;
          window.emberAutoImportDynamic = function(specifier) {
            if (arguments.length === 1) {
              return r("_eai_dyn_" + specifier);
            } else {
              return r("_eai_dynt_" + specifier)(Array.prototype.slice.call(arguments, 1));
            }
          };
          window.emberAutoImportSync = function(specifier) {return r("_eai_sync_" + specifier)(Array.prototype.slice.call(arguments, 1));};
          d("lodash-es/find", [], function() { return __webpack_require__(/*! lodash-es/find */ "./node_modules/lodash-es/find.js");});
          d("lodash-es/startsWith", [], function() {return __webpack_require__(/*! lodash-es/startsWith */ "./node_modules/lodash-es/startsWith.js");});
          d("_eai_dyn_lodash-es/concat", [], function() {return __webpack_require__.e(/*! import() */ "node_modules_lodash-es_concat_js").then(__webpack_require__.bind(__webpack_require__, /*! lodash-es/concat */ "./node_modules/lodash-es/concat.js"));});
        })();
      }),

-    "../../../../private/var/folders/q3/yzggf_l965zbzps_bk8xr37c0000gn/T/broccoli-4171121iPrUeBlpJ8/cache-307-webpack_bundler_ember_auto_import_webpack/l.js":
-      (function(module, exports) {
-        window._eai_r = require;
-        window._eai_d = define;
-      })
  });
  // ... webpack internals 
})();
```
</details>

> Note that ember-auto-import needs knowledge about the obfuscated names of the (still globally defined) functions (e.g.
> `__ember_12345_define` in this example) here. This should be exposed to privileged addons like ember-auto-import or
> Embroider *somehow*, but as this is a strictly private API, it is not detailed here.

</details>

---

#### EmberENV

Ember reads environment variables from a `EmberENV` hash to enable/disable certain runtime features. Usually this is set
from the app's `config/environment.js` and `@ember/optional-features`, and compiled into the top of `vendor.js`:

<details>
<summary>vendor.js</summary>

```js
window.EmberENV = (function(EmberENV, extra) {
  for (var key in extra) {
    EmberENV[key] = extra[key];
  }

  return EmberENV;
})(window.EmberENV || {}, {"FEATURES":{},"EXTEND_PROTOTYPES":{"Date":false},"_APPLICATION_TEMPLATE_WRAPPER":false,"_DEFAULT_ASYNC_OBSERVERS":true,"_JQUERY_INTEGRATION":false,"_TEMPLATE_ONLY_GLIMMER_COMPONENTS":true});
```
</details>

But as you can see there, you can also inject those by setting a globally defined `window.EmberENV`.

In `no-globals` mode though, the latter behavior will be disabled. Only the variables built into `vendor.js` are taken
into account. Implementation wise this can happen in the same way as described above, by defining `EmberENV` within the
IIFE (and not using the explicit `window.` namespace).

## How we teach this

The [Optional Features Guide](https://guides.emberjs.com/release/configuring-ember/optional-features/) already covers
how to enable optional features. The [Embedding Applications Guide](https://guides.emberjs.com/release/configuring-ember/embedding-applications/)
should mention the `no-globals` feature to prevent conflicts of globals.

## Drawbacks

* It adds complexity and inter-dependencies to the various pieces involved in the build system, when separate bundles
  (e.g. generated by Embroider or ember-auto-import) need to know what the still global(!) but obfuscated reference to
  *this* app's `define` and `require` aliases is. See the implementation details outlined above. The benefits (for Ember
  *users*) should still outweigh the drawbacks (for Ember *maintainers*).

## Alternatives

* `loader.js` has a `loader.noConflict()` API, that was apparently meant to prevent conflicts by reassigning the previously
  set values for the `define` and `require` globals. However, this come with several issues:
  * there is no convenient way to set this in an Ember app.
  * it requires further changes (https://github.com/ember-cli/loader.js/issues/204, https://github.com/ember-cli/ember-resolver/issues/629)
    to work at all.
  * fundamentally it is not able to guarantee the absence of conflicts, as it still makes use of the global namespace. It
    just reverts the previous global assignments, which means these change *over time*. When we cannot control the order
    of execution (e.g. 3rd party scripts that run after Ember has overridden the globals, but before that has been reverted,
    see https://github.com/ember-cli/loader.js/issues/204#issuecomment-780709127), there is a big enough time window where
    things can go wrong.

* We could go "all in" and not only prevent setting the AMD-like functions as globals, but completely get rid of them (when
  that optional feature is enabled) as a (semi-)public API. The compiled app could still make use of some or the same runtime
  module system *internally*, e.g. using obfuscated function names that are not available for user-land code anymore. This
  would *completely* support the use case of running apps inside node.js without conflicting instances of `require()`, see 
  the Electron example in the "Motivation" chapter. However, that would be a much more invasive step, as it would break 
  any (addon!) code that still uses these functions explicitly.

  Note that we could (should?) still get to this point *eventually* when adopting this RFC as-is, by deprecating usage of
  AMD-like APIs by user-land code and removing them through a second optional feature, to be proposed in subsequent RFCs.

* Keep things as they are, and highlight in Ember's docs that running an Ember app *safely* alongside another Ember app
  or 3rd party JavaScript on the same page is not guaranteed to work. The only absolutely conflict-free way to do so is
  to use an `<iframe>`, with all the UX/DX constraints that this inherently brings.


## Unresolved questions

None at this point.
