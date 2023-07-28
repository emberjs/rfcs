---
stage: accepted
start-date: 2023-07-19T00:00:00.000Z
release-date:
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - framework
  - learning
prs:
  accepted: https://github.com/emberjs/rfcs/pull/938
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

# Strict ES Module Support

## Summary

Deprecate `require()` and `define()` and enforce the ES module spec.

## Motivation

Ember has been using ES modules as the official authoring format for many years.
But we don't actually follow the ES modules spec. We diverge from the spec in
several important ways:

 - our "modules" can be accessed via the synchronous, dynamic `require()`
   function. They evaluate the first time someone `require`s them. This behavior
   is impossible for true ES modules, which support synchronous-and-static
   inclusion via `import` or asynchronous-and-dynamic inclusion via `import()`,
   but never synchronous-and-dynamic inclusion.

 - in the spec, trying to import a nonexistent name from another module is an
   early error -- your module **will never even execute** if it does this. In
   the current implementation, your code happily runs and receives an undefined
   local binding.

 - in the spec, these two lines mean very different things:
    ```js
    import * as TheModule from "the-module";
    import TheModule from "the-module";
    ```
    In our current implementation, you can sometimes use them interchangeably.

 - In the spec, modules are always read-only. You cannot mutate them from the outside. In our implementation, you can.

This is very bad, because
 - our code breaks when you try to run it in environments that actually follow the spec.  
 - refactors of Ember and its build pipeline that would otherwise be 100%
invisible to apps are instead breaking changes, because apps and addons are
relying on leaked details (including timing) of the AMD implementation.
 - we cannot implement other standard ECMA features like top-level await until these incompatible APIs are gone (consider: how is `require()` supposed to deal with discovering a transitive dependency that contains top-level await?)

## Detailed Design

### Architectural Intro

There are four salient layers to consider in Ember's architecture:

 - Container holds state. When Container is asked for something it doesn't have yet, it gets the factory from the Registry.
 - Registry holds factories. You can manually register factories with the registry. When someone asks for a factory that is not already registered, the Registry delegates to the Resolver.
 - Resolver maps between requests like "service:translations" and module names like "my-app/services/translations". How it does this mapping can be customized. After mapping the name, it retrieves a module from loader.js.
 - loader.js holds modules. Modules get added via `define` and retrieved via `require`. In addition to being used by the Resolver, loader.js is used directly whenever a module imports another module.

This RFC replaces loader.js with ES Modules. 
 - instead of using `define` to get modules into the system, you must create an actual module in your app or addon. Nothing about "how do I create a module" is changed by this RFC. It has long been possible to just create a javascript file and the default build system would `define` it for you. As long as you stick to that normal happy path, none of your code is changing. It will just go through some other mechanism than `define` to get to the browser. (Whether that mechanism is literally browser-native ES modules or some other transpilation target is immaterial -- the point is that if you follow the ES module spec it doesn't matter.)

 - instead of using `require`, modules can only access each other via `import` or `import()` (or `import.meta.glob()`, see [RFC 939](https://github.com/emberjs/rfcs/pull/939)).

This RFC does not immediately propose changing Container or Registry or the Registry-facing Resolver interface. It *does* imply that Resolver's implementation must change, because today Resolver relies on `require` to synchronously retrieve things from loader.js, which will not be possible.

Instead, anything that needs to be synchronously resolvable by the Registry will need to be either:
 - explicitly preloaded into the Registry (using the existing `register` public API)
    ```js
    import Button from './components/button';
    registry.register('component:button', Button);
    ```
 - or passed into a new Resolver implementation that has some access to **preloaded** ES modules:
    ```diff
    // app/app.js
    -import Resolver from 'ember-resolver';
    +import Resolver from '@ember/resolver';
    export default class App extends Application {
    -  Resolver = Resolver;
    +  // For illustration purposes only! Not real API.
    +  Resolver = Resolver(import.meta.glob('./**/*.js', { eager: true }))
    }
    ```
    This would continue to support existing custom resolver rules, but people would need to extend their custom resolvers from this new Resolver that serves requests out of preloaded modules, rather than extend from the current Resolver that serves them out of loader.js.

### New Feature: strict-es-modules optional feature

We will introduce a new Ember optional feature:

```json
// optional-features.json
{
  "strict-es-modules": true
} 
```

And we will emit a deprecation warning in any app that has not enabled this new optional feature.

When `strict-es-modules` is set:
 - the global names `require`, `requirejs`, `requireModule`, `define`, and `loader` become undefined.
 - importing anything from `"require"` becomes an early error. This outlaws usage like:
      ```js
      import require from "require";
      import { has } from "require";
      ```
 - Default module exports will no longer get automatically converted to namespace exports.
 - Importing (including re-exporting) a non-existent name becomes an early error.
 - Attempting to mutate a module from the outside throws an exception.
 - `importSync` from `'@embroider/macros'` continues to work but it no longer guarantees lazy-evaluation (since lazy-and-synchronous is impossible under strict ES modules). 
 - the new special forms `import.meta.EMBER_COMPAT_MODULES` and `import.meta.EMBER_COMPAT_TEST_MODULES` become available (see next section).


### New Feature: import.meta.EMBER_COMPAT_MODULES

In general, it's difficult and undesirable to make every app try to express (even through pattern matching) the full set of modules that would have traditionally been forced into the bundle and made available to the Resolver. The set encompasses not just the app itself but also the merged app trees of all addons, plus each addon's own module namespace.

Instead, we introduce a special expression `import.meta.EMBER_COMPAT_MODULES` that expands to that eagerly imported set, as an object.

```js
// this:
const modules = import.meta.EMBER_COMPAT_MODULES;
// expands to roughly:
import * as $0 from "my-app/app.js";
import * as $1 from "my-app/router.js";
// ...
import * as $50 from "some-addon/components/whatever.js";
// ...
const modules = {
  "my-app/app.js": $0,
  "my-app/router.js": $1,
  // ...
  "some-addon/components/whatever.js": $50,
  // ...
}
```

Classic builds have one implementation-defined answer for `EMBER_COMPAT_MODULES`, whereas Embroider adjusts the meaning of `EMBER_COMPAT_MODULES` depending on your settings. For example, `staticComponents:true` removes `app/components/**` from `EMBER_COMPAT_MODULES`.

`import.meta.EMBER_COMPAT_MODULES` is only valid within an app, it has no meaning if you try to use it in an addon.

As a corollary to `import.meta.EMBER_COMPAT_MODULES` we will provide `import.meta.EMBER_COMPAT_TEST_MODULES`. This is the set of modules that would have traditionally been forced into the tests bundle. It includes things like the app's tests plus addon-test-support trees. 

It's named "COMPAT" because we intend it as a backward-compatibility feature, enabling you to continue using pre-Polaris idioms or addons with pre-Polaris idioms. In a fully-Polaris app with only updated addons, you would not need this at all. All template resolutions would be strict, so they would never hit the resolver. Similarly we expect that the Polaris story around service-as-resource and router-composed-out-of-components-and-resources would eliminate the remaining resolver lookups.

By making this feature explicitly visible in the code, we allow early adopters to choose *not* to use it at whatever point they're ready for that, rather than continuing to make this behavior invisible.


### New Feature: Static Resolver

We propose this blueprint change, which would be required when using the `strict-es-modules` optional feature:

```diff
// app/app.js
-import Resolver from 'ember-resolver';
+import Resolver from '@ember/resolver';
export default class App extends Application {
-  Resolver = Resolver;
+  Resolver = Resolver.withModules(import.meta.EMBER_COMPAT_MODULES);
}
```

This new `Resolver` is intended to offer the same extensibility API as the current one from `ember-resolver`. The main difference is that instead of looking in loader.js, it looks in its own (private) set of known modules. It offers a static method that extends itself to include more modules:

```ts
class Resolver {
  static withModules(modules: Record<string, unknown>): this;
}
```

This is a static method because `Application` expects `Resolver` to be a factory, not an instance, and we don't need to change that.

We will also take this opportunity to make this new Resolver:
 - not extend from `Ember.Object`
 - not include any deprecated APIs from the previous Resolver
 - drop any API we discover during implementation that is only relevant on unsupported Ember versions. (For example, I can see `ember-resolver` is still looking for stuff in `Ember.TEMPLATES`. That is definitely not a thing anymore!)

Along with this change, the existing `ember-resolver` package should be marked deprecated.




## Transition Path


### Implication: No touching Ember from script context

Today, an addon can use the `app.import` API in ember-cli to inject a script into the build, and that script can access parts of Ember via `require`. That behavior is intentionally made impossible by this change.

Addon authors are encouraged to offer explicit module-based API instead, where apps are expected to import your special feature before importing anything else from Ember that you were trying to influence.

### Implication: eager importSync

The `importSync` macro:

```js
import { importSync } from '@embroider/macros'
```

exists primarily as a way to express conditional inclusion in places where Ember doesn't offer the ability to absorb asynchrony.

This continues to work under strict-es-modules, but we can no longer guarantee *lazy evaluation* of the module that you `importSync`. 

We still guarantee **branch elimination in production builds**, so if you say:

```js
import { importSync, macroCondition, getOwnConfig } from '@embroider/macros';

let implementation;
if (macroCondition(getOwnConfig().SOME_FLAG)) {
  implementation = importSync('./new-one');
} else {
  implementation = importSync('./old-one');
}
```

we continue to guarantee that in production builds your bundle will only ever contain one of those implementations and not both. But in development builds, both may be present *and evaluated* in your application.

And if you don't do branch elimination at all:

```js
import { importSync } from '@embroider/macros';

class Example extends Component {
  onClick = () => {
    let util = importSync('./some-utility').default;
    util();
  }
}
```

your dependency is effectively static, as if you had written:

```js
import * as _some_utility  from './some-utility';

class Example extends Component {
  onClick = () => {
    let util = _some_utility.default;
    util();
  }
}
```

This can still be useful in the case of template-string-literal `importSync`:

```js
import { importSync } from '@embroider/macros';

class Example extends Component {
  onClick = () => {
    let widget = importSync(`./widgets/${this.which}`);
    widget.doTheThing();
  }
}
```

because it will automatically build into your app all the possible matches, when you want that.

## Recommendations: Replacing `require` in Addons

- If you're trying to import Ember-provided API that is only available on some Ember versions, use `dependencySatisfies` to guard the `import()` or `importSync()`.

- If your addon is trying to access code *from the app*: no, sorry, don't do that! Libraries don't get to magically import code out of their consumers. Ask politely for your users to give you want you need.

   Often this provides better developer experience anyway. For example, [ember-changeset-validations](https://github.com/poteto/ember-changeset-validations/tree/master#overriding-validation-messages) will try to load `app/validations/messages.js` to find the user's custom messages. But if the API was made slightly more explicit:


   ```diff
   // app/validations/messages.js
   +import { addCustomMessages } from 'ember-changeset-validations';
   -export default {
   +addCustomMessages({
     inclusion: '{description} is not included in the list',
   -}
   +});

   // app/app.js
   +import "./validations/messages.js";
   ```

   it makes the purpose of the code clearer to readers, and it has the opportunity to provide better types.

   This also gives users control over *how much* of their app is actually subject to this configuration. If they want it to remain global they can import it from `app/app.js`. But if they only use it in some particular `/admin` route, all the configuration can happen there and the code for it can avoid ever loading on other routes.


## Recommendations: Replacing `require.has` in Addons

- If you offer optional features that should only be available when another package is present, it can be far less confusing and error-prone to ask users to configure it explicitly from their app:

    ```js
    // Example: configure the Widget component to use luxon for date handling, and share it for reuse throughout this particular application.
    import { Widget } from "my-fancy-widget";
    import * as luxon from 'luxon';
    export default Widget.withDateLibrary(luxon);
    ```

    This can be especially important when people are code-splitting their applications. They might very well want to only add the optional library *sometimes*. 

    It also aids migrations: your addon can be used both with and without the optional library on a case-by-case basis as people migrate between the two states.

    It also makes it possible for your TypesScript signature to reflect whether or not the optional feature is enabled.

- If you're trying to do version detection of another package, you can use `dependencySatisfies` from `@embroider/macros`. This is appropriate when you're trying to decide whether it would be safe to `import()` something from the other package. In this case the other package should be listed as an optional peerDependency of your package.

## Recommendations: Replacement APIs for `requirejs.entries`

(This section also covers `requirejs._eak_seen`, which is a *super* old synonym for `requirejs.entries` that dates back all the way to Ember App Kit circa 2015, yet continues to work today!)

- Some usages of `requirejs.entries` are really doing the same thing as `require.has` and the above section applies to them as well.

- For cases where enumerating modules is truly necessary and legitimate, we're offering up a companion RFC to this one introducing `import.meta.glob`. The biggest difference between `import.meta.glob` and `requirejs.entries` is that you can only `import.meta.glob` your own package's files. If an addon wants to enumerate files from the app, you need to ask the app author to pass you the `import.meta.glob()` results. See [the RFC](https://github.com/emberjs/rfcs/pull/939) for details.


## Recommendations: Replacement APIs for `define`

 - If you're using `define()` so that your users can import from a name other than your package name: Nope, sorry, never do that. You're rudely stomping on a piece of the package namespace that doesn't belong to you. Provide a regular ES module and tell users to import from your real package name. Or rename your package to match the public API you really want.

 - If you're using `define()` to provide something that Ember will resolve, instead of defining it as the module level you can still `register` it at a the Registry level:

    ```js
    getOwner(this).register('service:special', MyService);
    ```

    I mention this option because it's stable public API. But see the "Alternatives" section below for a discussion on why we think Registry and Container themselves may not be needed once Ember is fully oriented around ES modules (which goes beyond the scope of this one RFC).

 - If you're using `define()` to make a third-party library available for import inside Ember apps, stop doing that in favor of ember-auto-import. 

 - If you're using `define()` because you emit your code in `treeForVendor` rather than `treeForAddon`: move your code to `treeForAddon` and emit regular modules. If you think you need to be in vendor in order to run "early enough", you may need to tell your users to import your module from the beginning of their `app.js`. 


## How We Teach This

The "recommendation" sections above can be the actual content for the deprecation guides. That is the main teaching component of this RFC.

Ultimately this change results in less to teach, because we can just say "Ember uses ES modules" and lean on existing teaching materials for ES modules.

## Drawbacks

These APIs are widely-used, so this will definitely take work in:
 - Ember itself
 - the default set of addons that appear in the official blueprint
 - popular community addons

## Alternatives

### We could change the Registry and Container APIs too

The Registry and Container have APIs that are not really aligned with ES modules. They expect to lazily-and-synchronously evaluate modules the first time they're needed. If we were starting from scratch with ES-modules plus a good lifetime primitive (like `@ember/destroyable`) plus WeakMap, it's not clear that we would have a Container or Registry *at all*.

This RFC does not propose changing Container or Registry's public APIs right now. But that does mean that in the implementation of this RFC we need a build step that emits code to eagerly evaluate and register all the modules up front. That has always been *allowed* by the existing public APIs, and it's approximately what Embroider does internally already to make Ember apps work in strict ES module environments. 

I see three alternatives here:

1. Keep Registry and Container unchanged, accepting that we need to eagerly register everything they might need, no longer using their lazy resolution fallbacks.

2. Change their APIs to be asynchronous, deprecating the synchronous versions. This would snowball into other APIs like service injection too.

3. Eliminate them entirely, replacing all their use cases with pure ES modules.

Option 1 is what this RFC currently proposes, for ease of transition. Option 2 and 3 are probably equally disruptive, and I don't see a reason to keep Registry and Container around at all, so if we're going to make people change we might as well go all the way to Option 3 instead of Option 2.

To get to Option 3 there are several prerequisites:

 - it would require a new injection API, along the lines of:
    ```diff
    import { service } from '@ember/service';
    + import TranslationService from '../services/translation';
    class extends Component {
    -  @service translations;
    +  @service(TranslationService) translations;
    }
    ```

    This is something we probably want to do anyway to improve TypeScript support and to improve code splitting and encapsulation. But it would be easier to transition if that change was not a hard requirement of this change.

 - it would require deprecating non-strict templates (so GJS only)
   - and this requires deprecating the current router (which uses "bare templates", which cannot be gjs) in favor of a Polaris-generation router that is not yet designed.
  
 - it probably requires changes (or elimination) of initializers and instance-initializers

For all those reasons I don't think Option 3 is immediately viable. However, I do think we should ensure that all the changes we're asking the community to adopt are durable changes that prepare them for that world. The changes in this RFC are all things we can be confident are necessary.


## Unresolved Questions

Should we tackle FastBoot.require?
