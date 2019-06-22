- Start Date: 2019-06-21
- Relevant Team(s): Ember CLI, Ember.js, Learning
- RFC PR:
- Tracking:

# v2 Addon Format (Embroider Compatibility)

## Summary

This RFC defines a new package format that is designed to make Ember packages (meaning both apps and addons) statically analyzable and more compatible with the rest of the NPM & Javascript ecosystem. This RFC is the first step in stabilizing [Embroider](https://github.com/embroider-build/embroider) as our next-generation build system.

## Motivation

One of the good things about Ember is that apps and addons have a powerful set of build-time capabilities that allow lots of shared code with zero-to-no manual integration steps for the typical user. We have been doing “zero config” since before it was a cool buzzword (it was just called “convention over configuration”). And we’ve been broadly successful at maintaining very wide backward- and forward-compatibility for a large body of highly-rated community-maintained addons.

But one of the challenging things about Ember is that our ecosystem’s build-time capabilities are more implementation-defined than spec-defined, and the implementation has accumulated capabilities organically while only rarely phasing out older patterns. I believe the lack of a clear, foundational, build-time public API specification is the fundamental underlying issue that efforts like the various packaging / packager RFCs have tried to work around.

The benefits to users for this RFC (and Embroider in general) are:

- faster builds and faster NPM installs
- “zero-config import from NPM — both static and dynamic” as a first-class feature that works for both third-party libraries and Ember addons
- support for arbitrary code splitting
- tree-shaking of unused modules, components, helpers, etc from the app and all addons
- a layered build system with clearly documented APIs between the layers, so it's easier to experiment and contribute
- a build system that can take advantage of current and future investments by the wider Javascript ecosystem into code bundling & optimization.

## Key Ideas

### Fully Embrace ES Modules

Ember was one of the earliest adopters of ECMAScript modules, and Ember core team members were directly involved in helping move that features through TC39. Ember’s early experiences with modules influenced the spec itself. _Yet we have lagged in truly embracing modules._

For example, how do Ember apps express that they depend on a third-party library? The [app.import](https://ember-cli.com/user-guide/#javascript-assets) API. This should be ECMA standard `import`.

Another way to state the problem is that apps and addons all _push_ whatever code they want into the final built app. Whereas ES modules can _pull_ each other into the build as needed.

### Play nice with NPM Conventions

The ECMA module spec by itself doesn’t try to define a module resolution algorithm. But the overwhelmingly most popular convention is the [node_modules resolution algorithm](https://nodejs.org/api/all.html#modules_all_together).

Ember addons do respect node_module resolution for build-time code, but they do not respect it for runtime code. This is an unhelpful distinction.

### Verbose, Static Javascript as a Compiler Target

Ember’s strong conventions mean that many kinds of dependencies can be inferred (including _statically_ inferred) without requiring the developer to laboriously manage them. This is a good thing and I believe the current fad in the wider Javascript ecosystem for making developers hand-write verbose static imports for everything confuses the benefits of having static analysis (which is good) with the benefits of hand-managing those static imports (which is unnecessary cognitive load when you have clear conventions and a compiler).

This design is about compiling today’s idiomatic Ember code into more “vanilla” patterns that leverage ES modules, node_modules resolution, and spec-compliant static and dynamic `import` to express the structure of an Ember application in a much more “vanilla Javascript” way.

This compile step lets us separate the authoring format (which isn’t changing in any significant way in this RFC) from the packaging format (which can be more verbose and static than we would want in an authoring format).

# Detailed design

## Definitions

**package**: every addon and app is a package. Usually synonymous with “NPM package”, but we also include in-repo packages. The most important fact about a package is that it’s often the boundary around code that comes from a particular author, team, or organization, so coordination across packages is a more sensitive design problem than coordination within apps.

**app**: a package used at the root of a project.

**addon**: a package not used at the root of a project. Will be an **allowed dependency** of either an **app** or an **addon**.

**allowed dependency**: For **addons**, the **allowed dependencies** are the `dependencies` and `peerDependencies` in `package.json` plus any in-repo addons. For **apps**, the **allowed dependencies** are the `dependencies`, `peerDependencies`, and `devDependencies` in `package.json` plus any in-repo addons.

**Ember package metadata**: the `ember-addon` section inside `package.json`. This already exists in v1, we’re going to extend it.

**v2 package**: a package with `package.json` like:

    "keywords": [ "ember-addon" ],
    "ember-addon": {
      "version": 2
    }

**v1 package**: a package with `package.json` like:

    "keywords": [ "ember-addon" ]

and no `version` key (or version key less than 2) in **Ember package metadata**.

**non-Ember package**: a package without `keywords: ["ember-addon"]`

## Scope of this RFC

This RFC is intended as the base level spec for v2 Ember packages. **It does not attempt to cover everything a v1 package can do today**. For example, no provision is made in this RFC for:

- providing dev middleware
- providing commands and blueprints
- preprocessing your parent package's code
- modifying your parent package's babel config
- injecting content into index.html (contentFor)

It is understood that all of these are legitimate things for Ember addons to do. Defining these capabilities within v2 packages will be done in followup RFCs. It is simply too much scope to cover in one RFC.

Because we're hyper-focused on backward- and forward-compatibility, there is no harm in progressively converting some addons to v2 (which provides immediate benefits in terms of build performance and reduced fragility under Embroider) while others need to stay as v1 until we offer the features they need.

## Package Public API Overview

The format we are about to describe _is a publication format_. Not necessarily an authoring format. By separating the two, we make it easier to evolve the authoring format without breaking ecosystem-wide compatibility. The publication format is deliberately more explicit and less dynamic that what we may want for an authoring format.

First, here’s the list of things a v2 package can provide. More detail on each of these will follow:

- **Own Javascript**: javascript and templates under the package’s own namespace (the v1 equivalent is `/addon/**/*.{js,hbs}/`)
- **App Javascript**: javascript and templates that must be merged with the consuming app’s namespace (the v1 equivalent is `/app/**/*.{js,hbs}`). Other RFCs are working to move Ember away from needing this feature, but we are not gated on any of those and fully support **App Javascript**.
- **CSS**: available for `@import` by other CSS files (both in the same package and across packages) and by ECMA `import` directives in Javascript modules (both in the same package and across packages).
- **Implicit Dependencies**: scripts, modules, and stylesheets that should be implicitly included in the app or the app's tests whenever this addon is active. This is a backward-compatibility feature.
- **Renaming Rules**: allow a package to declare that some of its modules should be available at different import paths than their real, resolvable path. This is a backward-compatibility feature that new addons should not use.
- **Externals**: allows a package to declare that it imports some things that are not **allowed dependencies**. Instead of being resolved at build time, **externals** are deferred until runtime and get handled by the traditional `loader.js` `require()`.
- **Assets**: any files that must be available in the final built application directory such that they have public URLs (typical examples are images and fonts).
- **Build Hooks**: code that runs within Node at application build time. The v1 equivalent is an addon's `index.js` file.

## Own Javascript

The public `main` (as defined in `package.json`) of a v2 package points to its **Own Javascript**. The code is formatted as ES modules using ES latest features, meaning: stage 4 features only, unless otherwise explicitly mentioned elsewhere in this spec. (Addon authors can still use whatever custom syntax they want, but those babel plugins must run before publication to NPM.)

Templates are in hbs format. No custom AST transforms are supported. (Addon authors can still use whatever custom AST transforms they want, but those transforms must have already been applied before publication to NPM.)

Unlike v1 addons, there is no `/app` or `/addon` directory that is magically removed from the runtime paths to the modules. All resolution follows the prevailing Node rules.

The benefit of this design is that it makes our packages understandable by a broader set of tooling. Editors and build tools can follow `import` statements across packages and end up in the right place.

In v1 packages, `main` usually points to a build-time configuration file. That file is moving and will be described in the **Build Hooks** section below.

### Own Javascript: Imports

Modules in **Own Javascript** are allowed to use ECMA static `import` to resolve any **allowed dependency**, causing it to be included in the build whenever the importing module is included. This replaces `app.import`. Resolution follows prevailing Node rules. (This usually means the node_modules algorithm, but it could also mean Yarn PnP. The difference shouldn't matter if you are correctly declaring all your **allowed dependencies**.)

Notice that a package’s **allowed dependencies** do not include the package itself. This is consistent with how Node resolution works. To import files from within your own package you must use relative paths. This is different from how run-time AMD module resolution has historically worked in Ember Apps. (`@embroider/compat` implements automatic adjustment for this case when compiling from v1 to v2).

Modules in **Own Javascript** are allowed to use dynamic `import()`, and the specifiers have the same meanings as in static import. However, we impose a restriction on what syntax is supported inside `import()`. The only supported syntax inside `import()` is:

- a string-literal
- or a template string literal
  - that begins with a static part
  - where the static part is required to contain at least one `/`
  - or at least two `/` when the path starts with `@`.

This is designed to allow limited pattern matching of possible targets for your dynamic `import()`. The rules ensure that we can always tell which NPM package you are talking about (the `@` rule covers namespaced NPM packages, so you can't depend on `@ember/${whatever}` but you can depend on `@ember/translations/${lang}`). If your pattern matches zero files, we consider that a static build error. (For more background on the thought-process behind this feature, see [Embroider #198: Allow dynamic imports to accept more dynamic inputs](https://github.com/embroider-build/embroider/issues/198).

Modules in **Own Javascript** are allowed to import template files. This is common in today’s addons (they import their own layout to set it on their Component class). But in v2 packages, import specifiers of templates are required to explicitly include the `.hbs` extension.

### Own Javascript: Transpilation of imported modules

Any module you import, whether from an Ember package or a non-Ember package, gets processed by the app's babel configuration by default. This ensures that the app's `config/targets.js` will always be respected and you won't accidentally break your older supported browsers by importing a dependency that uses newer ECMA features.

There is an explicit per-package opt-out for cases where you're _sure_ that transpilation is not needed and not desirable. (See **Build Hooks** for details on the `skipBabel` option.)

## Own Javascript: Supported module formats for non-Ember packages

As already stated, V2 Ember packages must contain only ES modules. However, non-Ember packages in your **allowed dependencies** are allowed to contain ES modules _or_ CommonJS modules. This provides the best compatibility with general-purpose NPM utilities.

### Own Javascript: Macros

The V2 format deliberately eliminates many sources of app-build-time dynamism from addons. Instead, we provide an equivalently-powerful macro system and consider it an always-supported language extension (the macros are always available to every V2 package, ambiently, and we promise to give them their faithful build-time semantics).

See **Macro System** for the full details.

## App Javascript

To provide **App Javascript**, a package includes the `app-js` key in **Ember package metadata**. For example, to duplicate the behavior of v1 packages, you could say:

    "ember-addon": {
      "version": 2,
      "app-js": "./app"
    }

Like the **Own Javascript**, templates are in place in hbs format with any AST transforms already applied. Javascript is in ES modules, using only ES latest features. ECMA static and dynamic imports from any **allowed dependency** are supported. (Even though the app javascript will be addressable within the _app's_ module namespace, your own imports still resolve relative to your addon.)

By making `app-js` an explicit key in **Ember package metadata**, our publication format is more durable (you can rearrange the conventional directory structure in the future without breaking the format) and more performant (less filesystem traversal is required to decide whether the package is using the **App Javascript** feature.

## CSS

To provide **CSS**, a package can include any number of CSS files. These files can `@import` each other via relative paths, which will result in build-time inclusion (as already works in v1 packages).

If any of the **Own Javascript** or **App Javascript** modules depend on the presence of a CSS file in the same package, it should say so explicitly via an ECMA relative import, like:

    import '../css/some-component.css';

This is interpreted as a build-time directive that ensures that before the Javascript module is evaluated, the CSS file's contents will be present in the DOM. ECMA import of CSS files must always include the explicit `.css` extension.

> Q: Does this interfere with the ability to do CSS-in-JS style for people who like that?

> A: No, because that would be a preprocessing step before publication. It’s a choice of authoring format, just like TypeScript or SCSS. CSS-in-JS people would compile all their things to ES modules before we deal with it.

It is also possible for other packages (including the consuming application) to depend on a CSS file in any of its **allowed dependencies**, from either Javascript or CSS. From Javascript it looks like:

    // This will resolve the `your-addon` package and find
    // './some-component.css' relative to the package root.
    // The .css file extension is mandatory
    import 'your-addon/some-component.css';

And from CSS it looks like:

    @import 'your-addon/some-component';

What about SCSS _et al_? You’re still free to use them as your authoring format, and they should be transpiled to CSS in your publication format. If you want to offer the original SCSS to consuming packages, you’re free to include it in the publication format too. Since we’re making all packages resolvable via normal node rules, it’s now dramatically easier to implement a preprocessor that supports inter-package dependencies. (The same logic applies to TypeScript.)

## Implicit Dependencies

Within **Ember package metadata** we support several flavors of implicit dependencies:

- implicit-modules
- implicit-scripts
- implicit-styles
- implicit-test-modules
- implicit-test-scripts
- implicit-test-styles

Each one is a list of package-relative paths to files within the package.

Whenever your package is active, all of its implicit dependencies will be included in the build. The `-test-` variants will be included only in test suites, and the non-`-test-` variants are always included.

`implicit-modules` and `implicit-test-modules` mean that the app should be built as if someone has explicitly typed ECMA import statements for each of the listed modules.

`implicit-scripts` and `implicit-test-scripts` are for Javascript in a script context. Script context is different from module context (as defined by the ECMA spec). This is how an addon can push things into the equivalent of the traditioanl `vendor.js`, which is in script context.

`implicit-styles` and `implicit-test-styles` are for stylesheets. This is how an addon can push things into the equivalent of the traditional vendor.css.

While **Implicit Dependencies** are a fully-supported part of the v2 spec, v2 packages are encouraged to use direct ECMA `import` or CSS `@import` instead, whenever possible. A direct `import` provides finer-grained dependency information: we know exactly _which_ module inside your package actually depends on the thing, rather than needing to assume that your entire package depends on it.

For example, if one of your components depends on a third-party library, you should `import` that library directly from your component. Then the library will only be included if somebody uses that particular component. Whereas if you use `implicit-scripts`, the library will always be included, even if nobody uses the component that needs the library.

## Renaming Rules

V1 Addons have multiple ways (at least five that I've found so far!) of emitting modules that escape the addon's own package namespace. Examples:

- `ember-lodash` remaps all of its modules to the package name `lodash`
- `@ember/test-helpers` provides some modules under its true name, but also some modules under `ember-test-helpers`.

In order for Embroider compile these packages to valid V2 packages, we give V2 packages the ability to express renaming rules using the following properties in **Ember package metadata**:

- `renamed-packages`: a map from the package name a user would type to the real package name that provides it. Example:

  ```js
  {
    "renamed-packages": {
      "lodash": "ember-lodash"
    }
  }
  ```

- `renamed-modules`: a map from modules that a user may try to import to the real paths where those modules live:

  ```js
   "renamed-modules": {
    "ember-test-helpers/index.js": "@ember/test-helpers/ember-test-helpers/index.js"
   }
  ```

  When Embroider compiles a V1 package like `@ember/test-helpers` it ensures that the modules that would have "escaped" the package end up _inside_ the package, so that this kind of renaming works.

  The renaming rules allow these addons to adopt V2 format without breaking their public API. New addons _should not_ use renaming rules because it's confusing when the imports people type don't align with their real dependencies.

## Externals

The `externals` property in **Ember package metadata** allows a V2 addon to declare specific imported modules that should not be resolved at build time. Instead they will be resolved at runtime using the traditional `loader.js` `require()`.

When Embroider compiles V1 packages to V2 it does automatic externals detection.

When publishing a native V2 package, any externals need to be listed explicitly in **Ember package metadata**.

An example of when you may need `externals` is when you need to consume a script (not a module) that contains arbitrary `define()` statements. The modules defined by those statements aren't resolvable, in general, at build time. So attempts to import them will generate build errors. By listing them in externals, you can defer them until runtime where they will work.

## Assets

To provide **Assets**, a package includes the `public-assets` key in **Ember package metadata**. It's a mapping from local paths to app-relative URLs that should be available in the final app. For example:

    "name": "my-addon",
    "ember-addon": {
      "version": 2,
      "public-assets": {
        "./public/image.png": "/my-addon/image.png"
      }
    }

with:

    my-addon
    └── public
        └── image.png

will result in final build output:

    dist
    └── my-addon
        └── image.png

Notice that we’re _not_ choosing to include assets via explicit ECMA `import`. The reason is that fine-grained inclusion of asset files is not critical to runtime performance. Any assets that your app doesn’t actually need, it should never fetch. Assets are always things with their own URLs.

If two V2 packages try to emit assets with the same public URL, that's a build error.

> Q: Should we just automatically namespace them instead?
> A: That was considered, but it makes backward compatibility harder, and public URLs are not always free to choose/change.

## Build Hooks

In today’s v1 addon packages, the `index.js` file is the main entrypoint that allows an addon to integrate itself with the overall ember-cli build pipeline. The same idea carries forward to v2, with some changes.

It is no longer the `main` entrypoint of the package (see **Own Javascript**). Instead, it’s located via the `build` key in **Ember package metadata**, which should point at a Javascript file. `build` is optional — if you don’t have anything to say, you don’t need the file.

It is now an ECMA module, not a CJS file. The default export is a class that implements your build hooks (there is no required base class).

Here is a list of build hooks, each of which will have its own section below.

- configure
- configureDependencies
- skipBabel

I will describe the hooks using TypeScript signatures for precision. This does not imply anything about us actually using TypeScript to implement them. Each package has two type variables:

- `PackageOptions` is the interface for what options you accept from packages that depend on you. It's your package's build-time public API.
- `OwnConfig` is the interface for the configuration that you want to send to your own code, which your code can access via the `getOwnConfig` macro. This is how you influence your runtime code from the build hooks.

### Build Hook: configure

```ts
interface ConfigurationRequest<PackageOptions> = {
  options: PackageOptions,
  fromPackageName: string,
  fromPackageRoot: string,
};
configure<PackageOptions, OwnConfig>(
  requests: ConfigurationRequest<PackageOptions>[]
): OwnConfig
```

The configure hook receives an array of configuration requests. Each request contain the `PackageOptions` that a package that depends on this addon has sent to this addon. It also includes the `fromPackageName` and `fromPackageRoot` (the full path on disk to the requesting package) so that any configuration errors can blame the proper source.

`configure` deals with an array because multiple packages may depend on a single copy of our package. But our package can only be configured in one way (for example, we are either going to include some extra code or strip it out via the macro system, but we can't do both).

Addons are encouraged to merge configuration requests intelligently to try to satisfy all requesters. If it's impossible to do so, you can throw an error that explains the problem.

The `OwnConfig` return value must be JSON-serializable. It becomes available to your **Own Javascript** via the `getOwnConfig` macro, so that it can influence what code is conditionally compiled out of the build.

### Build Hook: configureDependencies

```ts
configureDependencies(): {
  [dependencyName: string]: PackageOptionsForDependency | "disabled"
}
```

The `configureDependencies` hook is how you send configuration down to your own dependencies. For each package in your **allowed dependencies** you may return either the `PackageOptions` expected by that package, or the string `"disabled"`.

Any dependencies that you don't mention are considered active, but don't receive and configuration from you.

Any dependency for which you provide `PackageOptions` is active, and will receive those `PackageOptions` in its own `configure` hook.

If you set a package to `"disabled"`, it will not become active _because of your addon_. It may still become active if another package depends on it and leave it active.

When and only when a package is active:

- all standard Ember module types (`your-package/components/*.js`, `your-package/services/*.js`, etc) from its **Own Javascript** _that cannot be statically ruled out as unnecessary_ are included in the build as if some application code has `import`ed them. (What counts as “cannot be statically ruled out” is free to change as apps adopt increasingly static practices. This doesn’t break any already published packages, it just makes builds that consume them more efficient.)
- all of the package's **Implicit Dependencies** are included in the build.
- all **App Javascript** is included in the build.
- all **Assets** are included in the build.
- the package's **Active Dependencies** become active recursively.
  ​​
  Whether or not a package is active:

- directly-imported **Own Javascript** and **CSS** are available to any other package as described in those sections. The rationale for allowing `import` of non-active packages is that (1) we follow node module resolution and node module resolution doesn’t care about our notion of “active”, and (2) `import` is an explicit request to use the module in question. It’s not surprising that it would work, it would be more surprising if it didn’t.

The `configureDependencies` hook is the _only_ way to disable child packages. The package hooks are implemented as a class with no base class. There is no `super` to manipulate to interfere with your children’s hooks.

### Build Hook: skipBabel

```ts
skipBabel({ package: string, semverRange?: string }[]): void;
```

By default, all imported dependencies (and their recursive importe dependencies) go through the app's babel config. This ensures browser compatibility safety. However, we provide `skipBabel` as an opt-out to work around transpilation problems in cases where the developer has verified that transpilation of a given package isn't needed.

`skipBabel` returns a list of packge names and optionally semver ranges. If no range is included, it defaults to `*`. This is a place where you're allowed to mentioned packages that are _not_ in your **allowed dependencies** because it may be necessary to talk about deeper dependencies within them. The `skipBabel` settings for all active addons are combined and if any addon skips babel for a given package & version, that causes the package to not be transpiled.

The semver range is useful to disambiguate if there are multiple versions of the same package involved in the app, and in cases where a developer has manually verified that transpilation isn't needed, it's good practice to use the semver range so that `skipBabel` doesn't accidentally apply to a future version of the package that may indeed need transpilation.

## What about Test Support?

v1 packages can provide `treeForTestSupport`, `treeForAddonTestSupport`, and `app.import` with `type="test"`. All of these features are dropped.

To provide test-support code, make a separate module within your package and tell people to `import` it from their tests. As long as it is only imported from tests, it will not be present in non-test bundles. (Things get simpler when you respect the module dependency graph.)

## Macro System

v1 packages can run arbitrary Node code that completely alters their runtime code. This makes them impossible to analyze. v2 packages are not allowed to do this. There are no "treeFor\*" hooks. Instead, they can influence their runtime code only through the macro system.

It helps to think about the macro system as an extension to Javascript itself that we allow in v2 packages. Because the macros are allowed to appear in any published V2 package, and because the macros are _not_ a dependency that each package can control (you don't get to bring your own separate macro system version with you), it's important that we design a small core that we can support for the long-term. We probably can't make breaking changes to the macro system, we can only introduce new macros.

(I'm proposing the macros live under `@ember/macros`. The current implementation of them lives in `@embroider/macros`.)

The Javascript macros are:

- importSync
- getOwnConfig
- getConfig
- macroCondition
- moduleExists
- dependencySatisfies
- failBuild

The Handlebars macros are:

- macroGetOwnConfig
- macroGetConfig
- macroCondition
- macroDependencySatisfies
- macroMaybeAttrs
- macroFailBuild

The difference in naming is because the JS macros get explicitly imported from `@ember/macros`, whereas the Handlebars macros do not, so they need an appropriate namespace prefix. (If we land template imports, I'm find with adjusting this RFC to make the names align.)

### JavaScript Macro: importSync

```js
import { importSync } from '@ember/macros';
importSync('some-dependency').default;
```

`importSync` exists to do a thing that standard Javascript does not do: synchronous dynamic import. Ember historically needs synchronous dynamic import (it's what our runtime AMD `require` does). Until some future date at which Ember has migrated away from synchronous module resolution we need `importSync`.

`importSync` is defined as behaving exactly like the standards-complient `import` except instead of returning `Promise<Module>` it returns `Module`.

In this RFC we don't state explicitly what `importSync` compiles to. It compiles to whatever the Javascript bundler we're using supports in order to achieve synchronous dynamic import. For example, if we're internally using Webpack we can compile `importSync("something")` to `require("something")`, because Webpack supports CommonJS `require` anywhere.

### JavaScript Macro: getOwnConfig

```js
// this example:
import { getOwnConfig } from '@ember/macros';
const shouldEnableCoolFeature = getOwnConfig().coolFeature;
// might compile to:
const shouldEnableCoolFeature = true;
// assuming your ownConfig is `{ coolFeature: true }`.
```

`getOwnConfig()` behaves like a function that returns your `OwnConfig`, as determined by the return value of your `configure` **Build Hook**. You're allowed to chain property accesses (including array indices) off of `getOwnConfig()`. Since the `OwnConfig` is required to be JSON-serializable, any subset of it can be accessed this way, and we inline that value directly into the code.

You can choose to inline the whole OwnConfig if you want to:

```js
// this example:
import { getOwnConfig } from '@ember/macros';
const myConfig = getOwnConfig();
// might compile to:
const myConfig = { coolFeature: true };
// assuming your ownConfig is `{ coolFeature: true }`.
```

Since `getOwnConfig` accesses the output of your `configure` build-hook, you have a place to run arbitrary

### Javascript Macro: getConfig

`getConfig` can access the `OwnConfig` of your dependencies.

```js
import { getConfig } from '@ember/macros';
const testSelectorsConfig = getConfig('ember-test-selector');
```

It supports property chaining the same as `getOwnConfig`.

This is a low-level power tool. It's mostly useful as a compile target for custom Babel plugins. For example, `ember-test-selectors` has a custom Babel plugin that _sometimes_ strips test properties out of your components. But if a V2 package is using ember-test-selectors, it needs to run the custom transform _before publishing_. At that point, it's too soon to decide whether to strip them.

Instead of actually doing the stripping, the ember-test-selector plugin would compile the user's code into code that uses `macroCondition` and `getConfig('ember-test-selectors')`. In this way, we get the powerful custo behavior, but only the standard core macros are needed at the time when the app itself is building.

### JavaScript Macro: macroCondition

`macroCondition` acts like a function that takes a boolean value and returns that same boolean value. But whenever `macroCondition` appears directly inside the predicate of an `if` statement or as the predicate of a ternary expression, it tells the macro system to do branch elimination based on the predicate. Here is an example that combines all three macros we've seen so far:

```js
// This example:
import { macroCondition, getOwnConfig, importSync } from '@ember/macros';

let implementation;

if (macroCondition(getOwnConfig().useNewVersion)) {
  implementation = importSync("./new-component");
} else {
  implementation = importSync("./old-component");
}

export default implementation;

// ==============
// Could compile down to this if OwnConfig contains { useNewVersion: true }
let implementation;
implementation = importSync("./new-component");
export default implementation;

// ===============
// Or compile down to this if OwnConfig contains { useNewVersion: false }
let implementation
implementation = importSync("./new-component");
export default implementation;
```

It is a build error if `macroCondition` cannot statically determine the truth status of its argument.

`macroCondition` supports boolean logic, like `macroCondition(getOwnConfig().a && getOwnConfig().b)`.

Here is an example of `macroCondition` in a ternary expression:

```js
const flavor = macroCondition(getOwnConfig().prefersChocolate) ? 'chocolate' : 'vanilla';
// could compile down to:
const flavor = 'chocolate';
```

`macroCondition` is the foundation that lets us choose which code to include in the build. You can choose to inline two different implementations within the branches of an `if` statement, or you can keep them in entirely separate modules and import only the correct one via `importSync`.

It would also be possible (in the future, when top-level await stabilizes) to use [top-level await](https://github.com/tc39/proposal-top-level-await) to replace usage of our `importSync` macro with standards-compliant `import()`:

```js
if (macroConditional(getOwnConfig().x)) {
  await import('x');
} else {
  await import('y');
}
```

Q: Why not allow `if (getOwnConfig().thing)` instead of `if (macroCondition(getOwnConfig().thing))`?

A: Because we don't want to leave any confusion over whether branch elimination will be done. Boolean expressions that include a macro like `getOwnConfig` alongside other runtime-only values are perfectly legal. But those expressions would not allow branch elimination. The ambiguity means you might accidentally defeat branch elimination without noticing. `macroCondition` is intended to signal -- both to the reader and to the compile -- that this place absolutely _must_ do branch elimination. It's an error if we can't eliminate one branch or the other.

### JavaScript Macro: moduleExists

Allow you to test if an `import` (or `import()` or `importSync()`, since they all accept an argument with identical semantics) would succeed. Always compiles to a boolean literal.

```js
import { moduleExists, macroCondition, importSync } from '@ember/macros';
if (macroCondition(moduleExists('ember-data'))) {
  const DS = importSync('ember-data').default;
  DS.Adapter.extend({
    //
  });
}
```

Remember that you're always only allowed to `import` from your own **allowed dependencies**. So if an addon wants to optionally use another package only if that package is present, that package must be listed as an **Optional Peer Dependency**.

### JavaScript Macro: dependencySatisfies

Allows you to test if the given **allowed dependency** satisfies the given semver range. Always compiles to a boolean literal.

```js
import { dependencySatisfies, macroCondition } from '@ember/macros';
if (macroCondition(dependencySatisfies('ember-data', '^3.0.0'))) {
  // include code here for ember-data 3.0 compat
}
```

The package version will be `semver.coerce`d first, such that non-standard versions like "3.9.0-beta.0" will appropriately satisfy constraints like "> 3.8".

### Javascript Macro: failBuild

Allow you to cause a build failure with a custom error message. If `failBuild` isn't eliminated by `macroCondition`'s branch elimination, the build will fail.

```js
import { dependencySatisfies, macroCondition, failBuild } from '@ember/macros';
if (macroCondition(dependencySatisfies('ember-data', '^3.0.0'))) {
  // include code here for ember-data 3.0 compat
} else {
  failBuild(`We don't support ember-data versions other than ^3.0.0`);
}
```

### Handlebars Macro: macroGetOwnConfig

`macroGetOwnConfig` is very similar to the `getOwnConfig` JS macro, but it works as a Handlebars helper. Given this `OwnConfig`:

```json
{
  "items": [{ "score": 42 }]
}
```

Then:

```hbs
<SomeComponent @score={{macroGetOwnConfig "items" 0 "score" }} />
{{! ⬆️compiles to ⬇️ }}
<SomeComponent @score={{42}} />
```

If you don't pass any arguments, you can get the whole thing (although this makes your template bigger, so use arguments when you can):

```hbs
<SomeComponent @config={{macroGetOwnConfig}} />
{{! ⬆️compiles to ⬇️ }}
<SomeComponent @config={{hash items=(array (hash score=42))}} />
```

### Handlebars Macro: macroCondition

Used as a helper within a block `{{#if}}` or inline `{{if}}`. Just like the JS `macroCondition`, it ensures that branch elimination will happen.

```hbs
  {{#if (macroCondition (macroGetOwnConfig "shouldUseThing")) }}
    <Thing />
  {{else}}
    <OtherThing />
  {{/if}}

  {{! ⬆️compiles to ⬇️ }}
  <Thing />
```

### Handlebars Macro: macroDependencySatisfies

Acts like a helper that returns a boolean. Like the `dependencySatisfies` JS macro, it can only resolve things that are **allowed dependencies**, so the same need for peer dependencies and/or **Optional Peer Dependencies** applies.

```hbs
<SomeComponent @canAnimate={{macroDependencySatisfies "liquid-fire" "*"}} />
{{! ⬆️compiles to ⬇️ }}
<SomeComponent @canAnimate={{true}} />
```

### Handlebars Macro: macroMaybeAttrs

There is one place where `{{#if}}` doesn't work: within "element space". If you want to _sometimes_ set an attribute, but sometimes not, this doesn't work:

```hbs
<div {{#if this.testing}} data-test-target={{@id}} {{/if}} />
```

`macroMaybeAttrs` exists to conditionally compile away attributes and arguments out of element space:

```hbs
<div {{macroMaybeAttrs (getConfig "ember-test-selectors" "enabled") data-test-target=@id }} />
```

It can be placed on both HTML elements and angle bracket component invocations.

### Handlebars Macro: macroFailBiuld

Like the JS `failBuild` macro.

```hbs
{{#if (macroCondition (dependencySatisfies "some-peer-dep" "^3.0.0")) }}
  <UseTheThing />
{{else}}
  {{macroFailBuild "You tried to use <MyFancyComponent/> but it requires some-peer-dep ^3.0.0"}}
{{/if}}
```

### Macros: Overall Design

All the macros are intended to be valid syntax. They shouldn't break parsing or linting.

While we guarantee that branch elimination will run in production builds, we _don't_ guarantee that in development. The macros are designed so that in development they may have _runtime_ implementations. This is powerful because it lets us produce a single build that works in multiple contexts. For example:

- it solves the longstanding problem that when you run your tests by visiting `localhost:4200/tests` the tests see the `development` environment, not the `test` environment. To get the test environment you can't use `ember serve`, you must use `ember test`. This has remained unfixed because it's expensive to do the whole build twice for the two environments.

  We can solve this problem by producing a _single_ build containing _both_ environments, guarded by the macro system. The macros can evaluate at runtime, allowing each environment to get the right thing. In production builds, test-only or dev-only branches will still be eliminated.

- it makes Fastboot builds simpler because we can guard the fastboot-only and browser-only code with the macro system. In development, we can run a single build that leaves both branches in and evaluates the macros at runtime.

The macros package (`@ember/macros` as proposed, `@embroider/macros` as implemented) will work in both regular ember-cli and in Embroider. And it will work in both V1 and V2 packages.

## Peer Dependencies

V2 packages can only resolve their **allowed dependencies**. This is fundamental rule that we can't break if we want the broadest compatibility with NPM and future compatibility with other strict systems such as [Yarn PnP](https://github.com/yarnpkg/rfcs/pull/101). Node often allows you to resolve things that are not **allowed dependencies** due to hoisting optimizations. But this is not safe or guaranteed, so we forbid relying on it.

This means that many things addons will try to access from their surrounding environment will need to be listed as `peerDependencies`. For example, addons that want to import `ember-data` should list `ember-data` as a `peerDependency`, so the app can control the `ember-data` version and the addon is guaranteed to resolve the same copy.

This also applies recursively -- if your addon wants to use an addon that needs `ember-data`, your addon should also list `ember-data` as a `peerDependency`. The clearest documented description of how recursive peerDependencies should work is in the [Yarn PnP Formal Guarantees](https://github.com/yarnpkg/rfcs/blob/master/accepted/0000-plug-an-play.md#c-formal-plugnplay-guarantees).

`ember-source` provides many "virtual" packages like `@ember/component`. If they were real packages, they would be `peerDependencies`, but having non-real packages in package.json is likely to result in errors. Pedantically, they can be listed in **externals** instead. In practice, they are a well-known set that we can always handle correctly automatically, so it's not very imporant whether an addon includes them in **externals**.

### Optional Peer Dependencies

Some addons optionally use another addon if it happens to be available in the app. In order to resolve such a dependency, we really need **Optional Peer Dependencies**.

NPM doesn't have a concept of optional peer dependencies. It has "optional dependencies", but they are something different and pretty useless.

Yarn did an [RFC for optional peer dependency support](https://github.com/yarnpkg/rfcs/blob/master/accepted/0000-optional-peer-dependencies.md). It is basically compatible with NPM, with the only caveat being that if you use NPM you may see a spurious warning at install time. As non-actionable peerDependency warnings are rife throughout the NPM ecosystem this doesn't seem like a big deal.

V2 Packages are allowed to use optional peer dependencies as described in the Yarn RFC.

Our own tooling, like ember-cli-dependency-checker, we can make sure the warnings respect the Yarn standard.

## Apps as V2 Packages

This RFC is focused heavily on addons, because that is the area that is most critical to standardize. Publishing addons to NPM in V2 format has major benefits:

- build performance: there is much less work to do at app build time, and many `dependencies` of addons can become `devDependencies` of addons, resulting in smaller `node_modules` and faster `npm install`.
- tool integration: VSCode, Typescript, SCSS, etc will all understand your code better when the dependencies are in V2 format. Things like "jump-to-definition" will work.
- Embroider stability: `@embroider/compat` needs to use heuristics and some addon-specific rules to compile V1 addons into V2. This is necessarily more fragile than having addons published natively in V2. The first step in stabilizing Embroider for mainstream adoption is standardizing on this new addon format.

In contrast, apps are not published to NPM. So where would they use V2 publication format?

During the build process for an app, it will first build from its authoring format _to the standard v2 package format_. At that point, the whole project is just a collection of standard v2 packages with well-defined semantics, and we can confidently treat that stage in the build pipeline as supported public API.

The benefit of this approach is that we can separately evolve authoring formats and last-stage packaging tools, while keeping a stable interface between them. The stable interface is designed to leverage general-purpose ECMA-spec-compliant features wherever practical, which makes it a rich target. For more detail on Embroider's three-stage build pipeline see [the README](https://github.com/embroider-build/embroider/blob/f5181d0d7eab146fd0dfcafdff552ee4fc129f2a/README.md#embroider-a-modern-build-system-for-emberjs-apps).

v2-formatted apps do differ in a few ways from v2-formatted addon, as described in the following sections.

### Features that Apps May Not Use

Several features in the v2 addon format are designed to be consumed _by the app_. These features aren’t appropriate in an app, because that is the end of the line — a v2-formatted app should be understandable by general-purpose Javascript tooling and have very little _implicit_ Ember-specific build semantics left.

Features that apps may not use include:

- the `implicit-*` keys in **Ember package metadata**.
- the `app-js` key in **Ember package metadata**
- the `build` key in **Ember package metadata**. We should consider updating the _authoring_ format so that apps can use a build file with the standard package hooks, because that makes a lot of sense. But it’s not appropriate in the v2 format (which is a _publication_ format), and this change can be a separate RFC, and it will be an easier RFC after landing this one.
- automatic inclusion of resolvable types (components, services, etc) from the **Own Javascript** of all **Active Dependencies** and the app itself.
- the `public-assets` key in **Ember package metadata**.

All these features can appear in v2 _addons_, and the _app_ ensures each one is represented by standards-compliant Javascript within the app’s own code. To illustrate with some examples, the V2 format for an app (as already implemented in Embroider) includes:

- `<script>` tag(s) in index.html and tests/index.html that ensure `implicit-scripts` and `implicit-test-scripts` of all active addons are alreay accounted for.
- `<link rel="stylesheet">` tag(s) in index.html and tests/index.html that ensure `implicit-styles` and `implicit-test-styles` are accounted for.
- actual Javascript `import` statements within the app's code that ensure `implicit-modules` and `implicit-test-modules` are accounted for
- actual Javascript `import` statements and AMD `define` calls that handle automatic inclusion of resolvable types that cannot be statically ruled out.

## Features that only Apps may use

There are also a few V2 package features only supported in apps. These are mostly of interest only to people working within ember-cli and/or embroider to implement new packaging tools. Each of these is a property in **Ember package metadata**:

- `rootURL`: has the same meaning as `rootURL` in `config/environment.js` in a standard Ember app.
- `assets`: a list of relative paths to files. The intent of `assets` is that it declares that each file in the list must result in a valid URL in the final app.

  The most important assets are HTML files. All `contentFor` has already been applied to them. (Remember, we’re talking about the publication format that can be handed to the final stage packager, not necessarily the authoring format.) It is the job of the final stage packager to examine each asset HTML file and decide how to package up all its included assets in a correct and optimal way, emitting a final result HTML file that is rewritten to include the packaged assets.

  Note that packagers must respect the HTML semantics of `<script type="module">` vs `<script>` vs `<script async>`. For example: don’t go looking for `import` in `<script>`, it’s only correct in `<script type="module">`

  File types other than HTML are allowed to appear in `"assets"`. The intent is the same (it means these files must end up in the final build such that they’re addressable by URLs). For example, a Javascript file in `"assets"` implies that you want that JS file to be addressable in the final app (and we will treat it as a script, not a module, because this is for foreign JS that isn’t going through the typical build system. If you actually want a separate JS file as output of your build, use `import()` instead). This is a catch-all that allows things like your `/public` folder full of arbitrary files to pass through the final stage packager.

  A conventional app will have an `"assets"` list that include `index.html`, `tests/index.html`, and all the files that were copied from `/public`.

- `template-compiler.filename`: the relative path to a module that is capable of compiling all the templates. The module exports :
  - `compile: (moduleName: string, templateContents: string) => string` that converts templates into JS modules.
- `template-compiler.isParallelSafe`: true if the template compiler can be used in other node processes
- `babel.filename`: the relative path to a module that exports the app's babel config.
- `babel.isParallelSafe`: true if the `babel` settings can be used in a new node process.
- `babel.majorVersion`: the version of babel the app's settings were written for. Only 6 and 7 are supported at this time.

Unlike addons, an app’s **Own Javascript** is not limited to only ES latest features. It’s allowed to use any features that work with the config in `babel.filename`. This is an optimization — we _could_ logically require apps to follow the same rule as addons and compile down to ES latest before handing off to a final packager. But the final packager is going to run babel anyway, so we allow apps to do all their transpilation in that final single pass.

## Compatibility Strategy

The `@embroider/compat` package exists to compile V1 packages to V2. This allows `@embroider/core` to always assume V2 packages as input, so we don't need to wait until every addon is natively available in V2 before we start getting the benefits of Embroider. However, there is still an incentive to convert as many addons as possible to V2, because they build faster and they will be more stable (the v1-to-v2 compilation isn't flawless, we need heuristics and package-specific rules to deal with some dynamic addon behaviors).

It also needs to be possible for an addon published as V2 to work in existing apps on existing ember-cli versions. This is enabled by:

- `ember-auto-import`, which serves as a high-fidelity polyfill for importing directly from NPM. V2 addons natively support importing from NPM, but they should depend on `ember-auto-import` so those imports will have the same meaning when used in classic ember-cli.
- `@ember/macros`, which shall provide correct semantics regardless of whether the package using them is published as V1 or V2 and regardless of whether the build is being done by classic ember-cli or Embroider. Native V2 packages under Embroider can alway use macros, without an explicit dependency on `@ember/macros`, but they should include the dependency so that macros will work in classic ember-cli.
- ember-cli already supports a `main` property in **Ember package metadata** and has supported it for many versions. This allows an addon to put its classic `index.js` file in a place other than the package's true `main`. This means that V2 addons can have their runtime `index.js` as `main`, and should point `ember-addon.main` to a `classic.js` file. The `classic.js` file should `require` and `export` a compatibility shim library that we will provide. The compatibility shim will have the classic methods like `treeForAddon`, `treeForPublic` that take the V2-formatted features and present them in a way that classic ember-cli will understand. Since V2 packages are much more static than V1 packages, this shim is expected to not be very complicated.

# How we Teach This

This RFC should have no direct impact on what app authors need to learn. They keep using addons the same way they always have. Future RFCs that take Embroider mainstream _will_ have impact, but that can be discussed then.

The impact on addon authors is more significant. This design is fully backward compatible, and the intention is that all existing addons continue to work (some with worse compatibility hacks than others in the v1-to-v2 compiler). But there will be a demand for addons published in v2 format, since it is expected to result in faster build times. My prediction is that people who are motivated to get their own app build times down will send a lot of PRs to the addons they’re using.

In many cases, converting addons to v2 makes them simpler. For example, today many addons use custom broccoli code to wrap third-party libraries in a fastboot guard that prevents the libraries from trying to load in Node (where they presumably don’t work). In v2, they can drop all that custom build-time code in favor of a macro-guarded `importSync`.

This design does _not_ advocate loudly deprecating any v1 addon features. Doing that all at once would be unnecessarily disruptive. I would rather rely on the carrot of faster builds and Embroider stability than the stick of deprecation warnings. We can choose to deprecate v1 features in stages at a later time.

# Alternative Designs

Embroider effectively supersedes both the [Packager RFC](https://github.com/ember-cli/rfcs/blob/master/active/0051-packaging.md) and the [Prebuilt Addons RFC](https://github.com/ember-cli/rfcs/pull/118). So both of those are alternatives to this one.

Packager creates an escape hatch from the existing ember-cli build that is supposed to provide a foundation for many of the same features enabled by this design. The intention was correct, but in my opinion it tries to decompose the build along the wrong abstraction boundaries. It follows the existing pattern within ember-cli of decomposing the build by feature (all app javacript, all addon javascript, all templates, etc) rather than by package (everything from the app, everything from ember-data, everything from ember-power-select, etc), which puts it into direct conflict with the Prebuilt Addons RFC.

The API that packager provides is also incomplete compared with this design. For example, to take the packager output and build it using Webpack, Rollup, or Parcel still requires a significant amount of custom code. Whereas taking a collection of v2 formatted Ember packages and building them with any of those tools requires very little Ember-specific code.

The prebuilt addons RFC addresses build performance by doing the same kind of work-moving as this design. Addons can do much of their building up front, thus saving time when apps are building. But it only achieves a speedup when apps happen to be using the same build options that addons authors happened to publish. This design takes a different approach that preserves complete freedom for app authors to postprocess all addon Javascript, including dead-code-elimination based on the addon features their app is using. The prebuilt addons RFC also doesn’t attempt to specify the contents of the prebuilt trees — it just accepts the current implementation-defined contents. This is problematic because shared builds artifacts are long-lived, so it’s worth trying to align them with very general, spec-compliant semantics.

# Supporting References

- There is a [SPEC draft](https://github.com/embroider-build/embroider/blob/master/SPEC.md) in the Embroider repo that predates this one, but covers a broader scope. Where this document contradicts SPEC.md, this document takes precedence and SPEC.md needs to be updated. But SPEC.md covers a broader scope, including the disposition of the other build hooks that will be handled in future RFCs.

- The definitive list of **Ember package metadata** fields is declared in [AppMeta and AddonMeta interfaces](https://github.com/embroider-build/embroider/blob/master/packages/core/src/metadata.ts). Each one is documented in an [Appendix in SPEC.md](https://github.com/embroider-build/embroider/blob/master/SPEC.md#appendix-list-of-ember-package-metadata-fields).
