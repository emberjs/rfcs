---
stage: ready-for-release
start-date: 2025-02-13T00:00:00.000Z
release-date:
release-versions:
teams:
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1075'
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

<-- Replace "RFC title" with the title of your RFC -->
# WarpDrive Package Unification

## Summary

Restructure the existing `@ember-data/*` and `@warp-drive/*` packages in order
to simplify the installation and configuration story while maintaining the benefits
of a multi-package architecture.

Required packages for a fully functional `ember.js` experience would be:

- `@warp-drive/core` (all the universal basics)
- `@warp-drive/ember` (reactivity and inspector integration, ember specific components)
- `@warp-drive/json-api` (cache implementation)

Additional packages that applications may choose to utilize would be:

- `@warp-drive/utilities`
- `@warp-drive/experiments`
- `@warp-drive/legacy`

A few packages would be deprecated, overall existing packages would remain and could still be individually installed and composed if desired.

## Motivation

WarpDrive is designed to be highly flexible and composable, with consuming applications able to
pick-and-choose combinations of packages alongside BYO/3rd-party implementations of interfaces.

WarpDrive packages generally try to follow a variation of the single-responsibility-principle:
while their surface area is larger than a single class or function, they tend to represent a
specific architectural boundary or concept and a group of tightly related primitives.

These are all good things, but it also makes the learning, onboarding and package.json maintenance
experience much harder once an application ejects from using the single `ember-data` package to
gain access to new features and abilities.

We want to rebalance the project, holding on to what has been great about small composable packages
while simplifying learning, onboarding and maintenance.

Now, as we rebrand to WarpDrive, seems an ideal opportunity to rebalance in this way as it gives us the opportunity to update names throughout.

## Detailed design

### The World Today

Currently, EmberData/WarpDrive publishes 18 primary source-code packages.

- `@ember-data/active-record`
- `@ember-data/adapter`
- `@ember-data/debug`
- `@ember-data/graph`
- `@ember-data/json-api`
- `@ember-data/legacy-compat`
- `@ember-data/model`
- `@ember-data/request`
- `@ember-data/request-utils`
- `@ember-data/rest`
- `@ember-data/serializer`
- `@ember-data/store`
- `@ember-data/tracking`
- `@warp-drive/build-config`
- `@warp-drive/core-types`
- `@warp-drive/ember`
- `@warp-drive/experiments`
- `@warp-drive/schema-record`

As well as the "meta" package which bundles together the needed dependencies and configures them for the "legacy" EmberData experience.

- `ember-data` 

We publish `mirror` and `types` packages for each of the primary 18 source-code repositories and the meta package (for a total of 57 published artifacts currently)

> [Info]
> Mirror packages allow for two-versions of the library to be used in a codebase
> simultaneously. Types packages allow older (pre 4.13/5.3) releases to consume
> the types from later releases as a way of getting a slightly better form of
> typescript support than what was offered by the DefinitelyTyped project.

We additionally publish a number of packages for tooling:

- `eslint-plugin-ember-data` (empty, reserved for security)
- `eslint-plugin-warp-drive`
- `warp-drive` (cli tool)

And as well we have multiple packages "under development", meaning they are currently unpublished but could become public in the future.

- `@warp-drive/holodeck`
- `@warp-drive/schema`
- `@warp-drive/diagnostic`

A final consideration: we plan to add packages for integrations with other ecosystems
as we expand the WarpDrive universe, these would be analogous to `@warp-drive/ember`

- `@warp-drive/{angular|vue|svelte|solidjs|etc}`

In short, lots of packages. So how do we clean this up and unify them?

### The World Tomorrow

We will introduce four new packages 🙈

- `@warp-drive/core`
- `@warp-drive/json-api`
- `@warp-drive/utilities`
- `@warp-drive/legacy`

Each of these four will also have a `mirror` (but not a `types`) equivalent. These four 
packages alongside the exising packages `@warp-drive/ember` and `@warp-drive/experiments`
for a total of **6** packages would become the **maximal** set of packages needing to be installed and configured by any Ember application.

For applications written with other frameworks, replace `@warp-drive/ember` with the ecosystem specific package and remove `@warp-drive/legacy` which can only be used with Ember for a maximum of 5 packages.

**None of the existing packages will cease to exist**

At least, not right away. A few of them will be restructured to support the the ideas outlined below describing what belongs in each of these 6, and two of them will be deprecated.

Retaining the actual package boundaries will allow us to continue to enforce strong boundaries between primitives, continue to offer applications looking to push boundaries the ability to compose an experience using a different combination of packages as desired, and give us the ability to bring new ideas into the library and move old ideas out of the library seamlessly.

But much like users of `ember-data` rarely had to consider where the source-code actually lived (it has not been in `ember-data` since 3.11), want future users to be able to get up and running with `warp-drive` with as few barriers as practical.

Lets look at what goes into each of the 6.

## @warp-drive/core

The primary package – what someone could install and use and need nothing else to achieve the most basic WarpDrive experience – is `@warp-drive/core`.

Unless otherwise specified, all packages moved into core are "clean" re-exports, meaning that the only thing necessary to change from the existing package to the new package is to change the package name and add the appropriate sub-path.

E.G.

```diff
-import { Type } from '@warp-drive/core-types/symbols';
+import { Type } from '@warp-drive/core/types/symbols';
-import { setConfig } from '@warp-drive/build-config';
+import { setConfig } from '@warp-drive/core/build-config';
```

This package is subject to the following constraints:

- it can contain and depend upon nothing framework specific
- it must contain everything required to setup and use a basic WarpDrive experience (managed requests)
- it should contain anything needed to go beyond the basic experience IF it would be usable by any framework and with any desired API. 

With this in mind: lets look at what packages move into core:

- `@warp-drive/core-types{/*}` => `@warp-drive/core/types{/*}`

This library which provides symbols and types that all (or most) other WarpDrive packages rely upon.

- `@warp-drive/build-config{/*}` => `@warp-drive/core/build-config{/*}`

This package provides our macros build-plugin configuration – allowing apps to utilize support for feature flags, deprecation removal, debug logging, and environment based debugging assistance. 

- `@ember-data/request` => `@warp-drive/core/request` (named exports only)

As the RequestManager is the heart of the basic experience, it will become a named import
from the root (see below). The remaining request utilities such as the promise cache and
request specific types will be available from the `/request` path.

We do not retain the `request/fetch` subpath (see next item).

- `@ember-data/request/fetch` => `@warp-drive/core`

The `Fetch` handler becomes a named import from the core path, this simplifies the imports when doing the most common configuration.

```diff
- import RequestManager from '@ember-data/request';
- import Fetch from '@ember-data/request/fetch';
+ import { RequestManager, Fetch } from '@warp-drive/core';
```

- `@ember-data/store{/*}` => `@warp-drive/core/store{/*}`

The lone exception in the store package is that the `Store` itself will become a named export in `@warp-drive/core`. We thinking giving `RequestManager` and `Store` the same level of precedence is important as the former is the heart of the basic experience and the latter the heart of the advanced experience.

- `@ember-data/graph{/*}` => `@warp-drive/core/graph{/*}`

Note: this package still does not have any public APIs, we believe we will be able to mark it public once schema-record reaches feature-complete status.

This package is in many ways a "utility" package for building a robust cache implementation offering ORM-like capabilities for use with the Store. As it is intended for use by any Cache implementation (not just the JSON:API cache) and because it is sufficiently advanced in ways Cache implementations are unlikely to want to attempt to replicate, we provide it from core and rely on tree-shaking to remove it if the cache implementation does not import and use it.

- `@ember-data/tracking` (gets deprecated)
- `@warp-drive/signals` (gets added)
- `@warp-drive/signals{/*}` => `@warp-drive/core/signals{/*}`

This move is the most chaotic one but we believe if we make it now it'll only affect the small number of apps that have manually configured all of the packages and their required peers, and we think we can do this move in a non-breaking way even for them.

Historically, `@ember-data/tracking` (and using TC39 terminology) this package has provided a `signal` and `computed` implementation as well as a batching mechanism for use by reactive primitives in other packages. By encapsulating our reactivity needs in this way, we've prepared ourselves to enable swaping out the underlying implementation as desired.

Today, this package uses `@tracked` and `@cached` as its implementations of signal and computed under the hood (handwave, its slightly more complicated than that), but we want to make the precise mechanism configurable.

We would keep the basic infrastructure, decorators and utils in a new `@warp-drive/signals` package which would then require being configured for a specific reactivity implementation (such as any of the many signals implementations available today).

We would move the ember-specific configuration code into the `@warp-drive/ember` package, and duplicate it in `@ember-data/tracking` with a deprecation to preserve existing behaviors.

- `@warp-drive/schema-record{/*}` => `@warp-drive/core/reactive`

This package enables applications to use deeply-reactive objects to access the data in the cache based upon a provided schema. It is the long-term replacement for `@ember-data/model`. We believe the SchemaRecord paradigm is flexible and powerful enough that even though we will retain the hook-based configuration for instantiating records we find it unlikely alternative record implementations will be built. By retaining the hook, should we (or someone else) decide to build an alternative, this code will be tree-shaken. That said, we believe this primitive is core to the WarpDrive experience.

## @warp-drive/json-api

This package will absorb the cache implementation from `@ember-data/json-api` (but not the request builders).

Every app that wants to go beyond the basic RequestManager experience and utilize a Store will need a Cache. We hope someday to see more cache implementations built, for now this is the only one officially available.

## @warp-drive/ember

This package stays the same but gains features from two other packages.

From `@ember-data/tracking` it will absorb the responsibility to configure the reactivity system to use ember's signals implementation (`@tracked`).

From `@ember-data/debug` it will absorb the repsonsibility of providing support for the ember-inspector, for as long as the library still integrates with ember-inspector. We may provide our own browser extension or "pluggable panel" in the future, at which point the proper home for extension support might get reconsidered. `@ember-data/debug` thus becomes deprecated.

Both of these changes mean that once deprecation cycles are complete, `@warp-drive/ember` becomes a required package for using EmberData/WarpDrive in an Ember application.

## @warp-drive/utilities

This package reconstitutes all or parts of four current packages:

- `@ember-data/rest/request{/*}` => `@warp-drive/utilities/rest{/*}`
- `@ember-data/active-record/request{/*}` => `@warp-drive/utilities/active-record{/*}`
- `@ember-data/json-api/request{/*}` => `@warp-drive/utilities/json-api{/*}`
- `@ember-data/request-utils{/*}` => `@warp-drive/utilities/request{/*}`

The primary exception is that the basic `CachePolicy` will move into core and be imported via

```ts
import { DefaultCachePolicy } from '@warp-drive/core/store';
```

We are making this move because we feel it is approaching the level of implementation that most apps will likely use it instead of authoring their own.

## @warp-drive/experiments

This package remains unchanged. It represents unstable experiments that we hope to eventually bring into the core experience.

## @warp-drive/legacy

This package exists somewhat-temporarily to make maintaining the legacy `ember-data` experience easier. It comprises of four packages:

- `@ember-data/adapter{/*}` => `@warp-drive/legacy/adapter{/*}`
- `@ember-data/serializer{/*}` => `@warp-drive/legacy/serializer{/*}`
- `@ember-data/model{/*}` => `@warp-drive/legacy/model{/*}`
- `@ember-data/legacy-compat{/*}` => `@warp-drive/legacy/legacy-compat{/*}`

## Configuring an Application

With all of the above changes in mind, here is what configuring an Ember application for
the recommended experience would look like:

**/app/services/store.ts**
```ts
import { RequestManager, Store, Fetch } from '@warp-drive/core';
import { CacheHandler, DefaultCachePolicy, SchemaService } from '@warp-drive/core/store';
import { JSONAPICache } from '@warp-drive/json-api';
import { instantiateRecord, teardownRecord, type SchemaRecord} from '@warp-drive/core/reactive';

import type { CacheCapabilitiesManager } from '@warp-drive/core/store/types';
import type { StableRecordIdentifier } from '@warp-drive/core/types';

export default class AppStore extends Store {
  requestManager = new RequestManager()
    .use([Fetch])
    .useCache(CacheHandler);

  cachePolicy = new DefaultCachePolicy({
    apiHardExpires: 5 * 60 * 1000 // 5min
    apiSoftExpires: 1 * 60 * 1000 // 1min
  });

  createSchemaService() {
    return new SchemaService();
  }

  createCache(capabilities: CacheCapabilitiesManager) {
    return new JSONAPICache(capabilities);
  }

  instantiateRecord(identifier: StableRecordIdentifier, createArgs?: Record<string, unknown>): SchemaRecord {
    return instantiateRecord(this, identifier, createArgs);
  }

  teardownRecord(record: SchemaRecord): void {
    return teardownRecord(record);
  }
}
```

**/ember-cli-build.js**
```ts
'use strict';

const EmberApp = require('ember-cli/lib/broccoli/ember-app');
const { maybeEmbroider } = require('@embroider/test-setup');

module.exports = async function (defaults) {
  let app = new EmberApp(defaults, {});

  const { setConfig } = await import('@warp-drive/build-config');
  setConfig(app, __dirname, {
    // the version of WarpDrive this file was generated with,
    // can be updated along with WarpDrive as long as deprecations
    // are resolved.
    compatWith: '5.4',
    debug: {
      // activate flags for logging here
    },
    deprecations: {
      // specify deprecated features to remove here
    }
  });

  return maybeEmbroider(app);
};

```

The below file would only be updated if `@warp-drive/utilities` was selected for use:

**/app/app.ts**
```diff
import Application from '@ember/application';
import compatModules from '@embroider/virtual/compat-modules';
import Resolver from 'ember-resolver';
import loadInitializers from 'ember-load-initializers';
+import { setBuildURLConfig } from '@warp-drive/utilities';
import config from './config/environment';

+setBuildURLConfig({
+  host: '/',
+  namespace: 'api',
+});

export default class App extends Application {
  modulePrefix = config.modulePrefix;
  podModulePrefix = config.podModulePrefix;
  Resolver = Resolver.withModules(compatModules);
}

loadInitializers(App, config.modulePrefix, compatModules);
```

**/package.json**
```diff
{
  "dependencies": {
+    "@warp-drive/core": "5.4.0",
+    "@warp-drive/ember": "5.4.0",
+    "@warp-drive/json-api": "5.4.0"
  }
}
```

**/tsconfig.json**

No changes are required to tsconfig, as we will ship types as stable when installing WarpDrive
via this mechanism. If for some reason we are unable to:

```diff
 {
   "compilerOptions": {
+     "types": [
+       "ember-source/types",
+       "@warp-drive/core/preview-types",
+       "@warp-drive/ember/preview-types",
+       "@warp-drive/json-api/preview-types",
+    ]
   }
 }
```

## Codemod / Lint Rules for the Migration

This can become the recommended experience for new apps without any codemods or lint rules
provided documentation is updated.

In the interest of moving folks to this experience: we should consider a lint rule with an autofix
that changes paths similar to when we first split ember and ember-data into their multi-package
architecture.

We should also consider a lightweight codemod that uninstalls any WarpDrive/EmberData packages that
are present and installs the appropriate packages from the set of 6, updating import paths in the process.

Likely we can safely automate this for any app using 4.13 or 5.3+. 

## Polaris / V2 App Blueprint

We would remove `ember-data` from the default blueprint and add the file changes above as shown. This takes
the place of an automated installer script (which we still want to do). We can replace these changes with
a script that prompts the user for selections once we have the tooling for doing so.

## Guides & ApiDocs

The guides will need to be updated to reflect the WarpDrive terminology and
new import locations. They are already in need of a refresh to align with modern
best-practices as we push towards delivering the Polaris experience for WarpDrive,
and this can be done all at once.

We will need a solution for ApiDocs. While the ApiDocs have been able to handle
multiple packages, we would want to document these APIs via their locations in the
new packages instead, which means dropping quite a lot of packages from being
contained in the ApiDocs which has url concerns.

The intent is to have tooling that "re-exports" docs from the original packages at
from the new home as well to avoid things "Store" being imported from "@warp-drive/core"
but documented via "@ember-data/store".

We could have that tool produce an artifact that contains information about both
locations for use by the docs: e.g. something along the lines of:

```ts
const classDoc = {
  name: 'Store',
  export: 'Store',
  module: '@warp-drive/core',
  location: 'packages/core/src/index.ts',
  upstream: {
    name: 'Store',
    export: 'default',
    module: '@ember-data/store',
    location: 'packages/store/src/index.ts',
  },
  tags: [],
  description: [],
  // ... etc.
}
```

## Lockstep Versioning

We will continue to publish these packages in lockstep with each other. Specifically this means that
peers and dependencies are pinned to the version(s) published in the same release.

We will also continue to follow the general rule of thumb we've had with WarpDrive/EmberData packages
to this point: newer packages begin at 0.0.0, progress to 0.X.0 when mostly stable, and then jump to
match the overall project version once stable. This way even new packages come to have identical
versions once stability is reached, making version management easier.

## Future Deprecation cycles

This package configuration provides an avenue for cleaner deprecation cycles in the future for major
concepts.

When something at the package-level of scope is removed from the core experience, we can do a two-stage
deprecation.

In stage-1, the imports in `/core` are deprecated and instead users must install and import from `/legacy`.
In stage-2, the feature in `/legacy` is deprecated. This would spread this sort of "concept removal"
deprecation across two majors by design, enabling us to keep our preferred path of long-tail deprecation
support while still signifying in a clear way which patterns are the current happy path.

## Drawbacks

As far as I am aware, nearly everyone has expresed a desire to simplify the config in *some* way, though ideas on how have varied.

If there is a reason to *not* do this it is about future-unknowns.

We learned with `ember-data` just how hard it could be to remove bad concepts from the library and architecture - a pain which drove us to split into packages for easier isolation, iteration and replacement in the first place.

By recombining *at all* we risk having this happen again as these packages will be broader in focus than what we have arrived at today. However, we do not intend to drop the use of individual packages, but rather recombine their exports into an ideal place for consumers, so I suspect this risk is comparatively low.

## Alternatives

The impact of not doing this will be a cost payed by both maintainers and consumers. Maintainers will have to write extra tooling to help consumers maintain their projects, and spend more time debugging and answering questions around installation and configuration.

Consumers will feel frustrated with not being able to quickly get an updated version, or working installation.

An alternative is to return to a single package. In addition to `ember-data`, the team also owns the `warp-drive` npm package. This would enable us to do something along the lines of re-exporting every package under a sub-path. That would look something like this:

```ts
warp-drive
  /store
  /request
  /request-utils
  /tracking
  /adapter
  /serializer
  /schema-record
  /legacy-compat
  /model
  /ember
  /serializer
  /debug
  /build-config
  /core-types
  /json-api
  /rest
  /active-record
  /experiments
```

We feel this approach comes with several drawbacks.

First, it makes it harder to understand which packages are part of the recommended experience, vs which are being phased out.

Second, it means that code from all packages is available for import and intellisense, increasing the odds of developers making mistakes. Often those mistakes are easy to miss like importing the same token (`findRecord`) from legacy-compat or active-record instead of json-api.

Third: with the rise in AI assisted coding, having those packages and types present in a codebase also increases the risk of AI suggesting code utilizing them.

Lastly: we think we can use this opportunity to re-organize the mental model and reduce the number of concepts and terms developers need to be aware of when using WarpDrive.

In short, the original motivating factors for splitting into many-packages instead of one-package remain unchanged.

## Unresolved Questions

Are there more import paths that we should shift the locations of?

For instance, these two types will be imported by nearly every store configuration:

```ts
import type { CacheCapabilitiesManager } from '@warp-drive/core/store/types';
import type { StableRecordIdentifier } from '@warp-drive/core/types';
```

Many types in `core/store/types` are specific to the store and are store-specific
variations of signatures in `core/types`, but there are a few exceptions to this
and `CacheCapabilitiesManager` is one of them. Maybe it and a few others make the move.

Similarly, the types for `Document/RecordArray/Collection` etc come from the store today
(and are then repurposed for `HasMany` and similar), but these are types for reactive
objects, and as such it may be best to move them into `/reactive`.

Because types support is still canary for WarpDrive, we do not have to answer this question
via RFC and can answer it through iteration. But if we find other non-type imports that
make sense to move we should call them out here.