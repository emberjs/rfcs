---
stage: released
start-date: 2023-10-06T00:00:00.000Z
release-date: 2025-10-15
release-versions:
  ember-cli: 6.8.0
teams:
  - cli
  - data
  - framework
  - learning
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/977'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/1062'
  released: 'https://github.com/emberjs/rfcs/pull/1147'
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

# v2 App Format 

## Summary

This RFC defines a new app format, 
building off the prior work in [v2 Addon format](https://rfcs.emberjs.com/id/0507-embroider-v2-package-format),
and is designed to make Ember apps more compatible with the rest of the JavaScript ecosystem. This RFC will define conventions of the app and clearly identify functionality that is considered "compatibility integrations" and could be considered optional in the near future.

## Motivation

When ember-cli was created there was no existing JS tooling that met the needs of the Ember Framework. Over the years we have added more and more developer-friendly conventions to our build system that many Ember applications and addons depend on. As the wider JavaScript tooling story has evolved over the years Ember has fallen behind, this is mainly because our custom-built tools have not been keeping up with the wider community and we haven’t been able to directly use any more advanced tooling in our apps. Efforts have been started to improve the situation with the advent of [Embroider](https://github.com/embroider-build/embroider) but the current stable release of Embroider still runs inside ember-cli and is somewhat bound in terms of performance and capability by the underlying technology [broccolijs](https://github.com/broccolijs/broccoli) and some other architectural decisions.

Over the past year the Ember Core Tooling team have been working hard to invert the control between bundlers and ember-cli, which means that instead of ember-cli running a bundler (such as Webpack in the current Embroider stable version) as part of its build system the whole Ember build process will essentially become a plugin to the bundler. This means that we can more effectively make use of bundler innovations, performance improvements, and we are more capable of adapting to whatever next generation of build systems come to the Javascript ecosystem.

With the Ember build system as a plugin to bundlers we have the ability to only intervene on things that are Emberisims (i.e. not-standard) and as we work to make Ember more standard we can eventually turn off these ”compatibility plugins”. Compatibility plugins will need to be powered by an “ember prebuild” that collects information about your app and addons and output metadata that the bundler plugins will consume. The intention is that this prebuild will be done automatically for you as part of the bundler plugin setup and you should not have to worry about preforming extra steps.

This RFC is not going to describe a new blueprint where you don't need any compatibility plugins (or an ember prebuild) to run an Ember app, this RFC instead is going to propose a new blueprint that has all the compatibility plugins turned on so that it is easiest for most people to upgrade to. Any discussion about a minimal “compatibility-free” blueprint should happen in a later RFC. 

## Key Ideas

Much like the [v2 Addon format RFC](https://rfcs.emberjs.com/id/0507-embroider-v2-package-format#key-ideas) we want the new app blueprint to rely on ES Modules and not leave anything hidden or automatic. This not only makes it easier for developers to know where things are coming from, it also makes it easier for bundlers to know what to do with Ember applications.

Each of the following sections go into detail of all the changes between the current blueprint and the new proposal which is currently being developed at https://github.com/embroider-build/app-blueprint

### Focus on Vite

The new app blueprint will default to using Vite to power local development and build for production. Vite has been on a meteoric rise in popularity over the past few years, and it represents the state of the art when it comes to Developer Ergonomics.  Standardising on such a popular build system will bring the Ember community a lot of benefits but the key areas of improvement that we expect developers to experience are: 

- significantly improved rebuild speeds during local development
- the ability to use "standard" vite plugins with your Ember apps
- significant improvements to debugability of your code in development because Vite serves ES Modules directly to your browser in development

### Removal of requirejs support

The default build system that will be used in this new blueprint will be Vite and for the most part Vite requires that your app is described in terms of ES Modules. The default build from `ember-cli` would express all of your modules using AMD and requirejs `define()` statements. There is an open RFC for [Strict ES Module Support](https://github.com/emberjs/rfcs/pull/938) that removes Ember's reliance on AMD to define modules and that RFC would be a requirement for this one i.e. when building with Vite you essentially have no requirejs module support. This means that any addon or application that interacts with requirejs would need to be updated for people to upgrade to this new blueprint.

### Top level await added

Since we are now using ES Modules througout the whole application it now becomes possible to use [top-level-await](https://v8.dev/features/top-level-await) in the module-scope of any of your files. This opens up a lot of possibilities for Ember developers and is a great example of how adopting standards helps to bring improvements from the wider ecosystem to Ember developers.

## Detailed design

In this section I'm going to go through each of the changes in the new proposed blueprint, in each section I will do my best to explain the reasoning for why it needs to be like this but if you have any questions or comments please feel free to comment on the RFC and we will try to update that section.

### Entrypoint -> index.html

A lot of modern bundlers make use of your index.html file as the main entrypoint to kick off bundling. Any files that are referenced in your index.html would then end up getting bundled.

This already an issue for an Ember app because the index.html that is traditionally found at `app/index.html` is ultimately not used directly. Information in the index.html file is used to generate the real file that is used at the end of the pipeline. 

This new blueprint proposes to remove this oddity and both allow bundlers to look directly at the source index.html instead of the output build artifact and will remove almost all ember build customisations that targed the index.html. We still intend to support `{{content-for}}` entries in the index.html with a few caveats that I will explain in more detail in the next section, but the support for `{{content-for}}` will need to be provided by a bundler plugin that has the ability to replace content in the index.html. 

The index.html will also have an inline script tag that performs the application boot. This process used to be hidden away in a `{{content-for “app-boot”}}` section in one of the javascript files automatically added to the index.html during the build pipeline. This explicit inline application boot significantly improves the visibility of how the application actually boots and should allow developers to customise that process without our build system needing to provide a way to handle those customisations. Incidently this now means that if you are using an addon to customise the `{{content-for “app-boot”}}` then this will no longer work. If Embroider discovers that an app is trying to customise the app-boot in any way it will throw this error: 

> Your app uses at least one classic addon that provides content-for 'app-boot'. This is no longer supported.
> 
> With Embroider, you have full control over the app-boot script, so classic addons no longer need to modify it under the hood.
> 
> The following code is used for your app boot:
> 
> {{inline-custom-app-boot-code}}
> 
> 1. If you want to keep the same behavior, copy and paste it to the app-boot script included in app/index.html.
> 2. Once app/index.html has the content you need, remove the present error by setting "useAddonAppBoot" to false in the build options.

The embedded boot section in the index.html will look like the following: 

```
<script type="module">
  import Application from './app/app';
  import environment from './app/config/environment';
  Application.create(environment.APP);
</script>
```

This boot section touches app/app.js and the new app/config/environment.js files which are both described in sections below.

#### content-for in index.html

The way that `{{content-for}}` works in ember-cli currently is under-specified and cronically under documented. The highest level summary is that anyone can currently add `{{content—for “any-string“}}` and as long as any active ember-addon provided a value for that specific content-for section the exact string that the addon provided would be injected at that point in the document. While we are familiar with sections like `{{content-for “head”}}` there is no pre-defined list and the common sections are only common due to convention. This makes it **extremely** hard for a modern build system that doesn’t understand ember-addons to be able to know what to do with these sections. 

We are proposing that we codify the conventional sections as the default set, and the ember prebuild will be able to collate the text each addon wants to add to these sections 

- head
- test-head
- head-footer
- test-head-footer
- body
- test-body
- body-footer
- test-body-footer
- config-module
- app-boot

If you are using any other custom `{{content-for}}` section then you will need to explicitly pass this to your embroider config via a `availableContentForTypes` configuration

### App Entrypoint -> app/app.js

In a classic build ember-cli would collect all the of the modules in your app into an entrypoint in the build that would go through and define each of the modules in the require.js AMD registry. There is already an [RFC that describes the fact that we want to deprecate this AMD registry](https://github.com/emberjs/rfcs/blob/strict-es-module-support/text/0938-strict-es-module-support.md), but the key thing for this RFC is that we are trying to think of our app as series of real ES Modules we need to provide some way for the built-in discovery of these modules that allows Ember to still resolve those modules by name (for Dependency Injection concerns like services)

We are providing this with the virtual module `'@embroider/virtual/compat-modules';`. This means that any bundler plugin that wants to support Ember needs to be able to support virtual module imports. The contents of this file can be obtained by asking the Embroider resolver which collates the list of modules during the Ember prebuild.

We then pass the list of modules to an updated version of the Ember Resolver (that has already been released) which means that the Ember Resolver no longer needs to rely on requirejs.entries to find all of the parts of your application. This set of compat modules also needs to be passed into the `loadInitializers()` from `ember-load-initializers` so that doesn't use AMD for initalizer discovery either. Here is an example of the difference in the `app/app.js` file:

```diff
 import Application from '@ember/application';
+import compatModules from '@embroider/virtual/compat-modules';
 import Resolver from 'ember-resolver';
 import loadInitializers from 'ember-load-initializers';
 import config from 'current-blueprint/config/environment';

 export default class App extends Application {
   modulePrefix = config.modulePrefix;
   podModulePrefix = config.podModulePrefix;
-  Resolver = Resolver;
+  Resolver = Resolver.withModules(compatModules);
 }

-loadInitializers(App, config.modulePrefix);
+loadInitializers(App, config.modulePrefix, compatModules);
```

One very important thing to note is that because we're moving from AMD to real ESM modules we now have a timing change for any code that is executing in the module scope, this code will now be executed eagerly and will not wait until the module is actually consumed by your app. This change happens because AMD inherently lazily executes module code only when that module is consumed, and ES modules are inherently a static graph of modules.

For the most part this change should not affect many people and most of the problems that we noticed related to this change came from addons that had code that had errors in it but was never cleaned up. It was never noticed before because the code was not being exercised by tests or consumers of the addons so most of the time the fix was to just delete the previously inert code.

### Application Config -> app/config/environment.js and config/environment.js

In a Classic build ember-cli automatically generated a module `app/config/environment.js` in your application build pipeline that is customisable with the config `storeConfigInMeta`. If `storeConfigInMeta` is true (which is the default) then the contents of this module will look for a `<meta>` tag in your html and parse out the your previously serialised config object and return the value as the default export from the module. This is what people recieve when they import from `app-name/config/environment`.

This is another example of an Ember-specific complexity in the build system that can be confusing for other build systems. In the new blueprint we propose making the `app/config/environment.js` file exist in your source, and the contents will clearly be loading the config from meta: 

```js
import loadConfigFromMeta from '@embroider/config-meta-loader';

export default loadConfigFromMeta('app-name');
```

The serialising of the config into the index.html is still being handled by `{{content-for 'head'}}` and is not going to change as a result of this RFC.

One restriction in the new blueprint is that we don't have an automatic implementation for when `storeConfigInMeta` is set to `false`. Our reasoning is that if you have set this setting then you are likely doing something custom and you would need to update `app/config/environment.js` to reflect your custom setup. We don't need to provide any customisation here because this is a user-owned module and you can edit it as you please.

### Test Entrypoint -> tests/index.html and tests/test-helper.js

The `tests/index.html` will have the same treatment as the main `index.html` where the test boot code will now be exposed directly in an inline script so that test booting is not hidden deep in the build pipeline.

```html
<script type="module">
    import { start } from './test-helper';
    import.meta.glob("./**/*.{js,gjs,gts}", { eager: true });
    start();
</script>
```

To facilitate this new API the test-helper needs to be changed to essentially "wrap" its contents in a function that can be called rather than it running as a side effect of the import: 

```diff
 import Application from 'current-blueprint/app';
 import config from 'current-blueprint/config/environment';
 import * as QUnit from 'qunit';
 import { setApplication } from '@ember/test-helpers';
 import { setup } from 'qunit-dom';
-import { start } from 'ember-qunit';
+import { start as qunitStart } from 'ember-qunit';
 
+export function start() {
 setApplication(Application.create(config.APP));
 setup(QUnit.assert);
 
-start();
+qunitStart();

+}
```

This also allows us to load qunit, load all the test files, and then start qunit once all the tests are loaded. The loading of the test files is now also made explicit with the line: 

```js
import.meta.glob("./**/*.{js,gjs,gts}", { eager: true });
```

`import.meta.glob` is described in detail in the [Introduce a Wildcard Module Import API RFC](https://rfcs.emberjs.com/id/0939-import-glob). It is natively supported in Vite but it would need to be implemented in any other build system that wants to support building an Ember app.

### Explicit Babel Config -> babel.config.cjs

In a classic build ember-cli-babel manages all configurations to babel in a way that is entirely hidden to the end-user. This is nice considering that users don't need to manage this file themselves, but it is also problematic because if anyone wants to customise their babel config they need to rely on extension points provided by both ember-cli and ember-cli-babel and in some cases those extension points may not even be available e.g. it is currently impossible for an app to configure an ast-transform for the ember-template-compiler and people workaround this issue by creating an in-repo addon that does this configuration for them.

In the new blueprint we will have an explicit `babel.config.cjs` that will come pre-configured with all the babel-plugins that ember-cli-babel would have configured for you.

Here is the full contents of the proposed babel file: 

```js
const {
  babelCompatSupport,
  templateCompatSupport,
} = require('@embroider/compat/babel');

module.exports = {
  plugins: [
    [
      'babel-plugin-ember-template-compilation',
      {
        compilerPath: 'ember-source/dist/ember-template-compiler.js',
        enableLegacyModules: [
          'ember-cli-htmlbars',
          'ember-cli-htmlbars-inline-precompile',
          'htmlbars-inline-precompile',
        ],
        transforms: [...templateCompatSupport()],
      },
    ],
    [
      'module:decorator-transforms',
      {
        runtime: {
          import: require.resolve('decorator-transforms/runtime-esm'),
        },
      },
    ],
    [
      '@babel/plugin-transform-runtime',
      {
        absoluteRuntime: __dirname,
        useESModules: true,
        regenerator: false,
      },
    ],
    ...babelCompatSupport(),
  ],

  generatorOpts: {
    compact: false,
  },
};
```

You can see that there are two functions being imported from `@embroider/compat/babel`: `babelCompatSupport()` and `templateCompatSupport()`. This collects any extra babel config that is provided by any installed v1 ember-addon and makes sure that it still works with this new config. When an app no longer has any v1 ember-addons these functions can be removed but we will likely be leaving them in the default blueprint for the foreseeable future because they cost nothing if they are not being used.

For people who are familiar with Babel config files you may have noticed that we have not included `@babel/preset-env` in this config. While our browser support has classically been handled by babel and `@babel/preset-env`, Vite has a config option [`build.target`](https://vite.dev/config/build-options#build-target) that controls what browsers you would like to down-compile your code for. This `build.target` config is passed to esbuild which is ultimately in charge of making sure your code runs in your stated targets i.e. in your `config/targets` file. While we are still going to read your `config/targets` file and pass that to the `build.target` config, it's worth noting that esbuild does not support the same range of browsers that `@babel/preset-env` does. This isn't a problem for the default blueprint because it supports all browsers that are part of Ember's official support matrix. We also don't see this as a blocker for adoption of this new blueprint because any applications that have a wider browser support matrix than Vite's `build.target` can provide can just manually configure `@babel/preset-env` in their babel config, now that it's significantly easier to edit your babel config.

### Ember Pre-Build Config -> ember-cli-build.js

To enable the current stable version of embroider you need to wrap your Ember Application defined in `ember-cli-build.js` in a `compatBuild()` function. The `compatBuild()` function takes a plugin that runs your bundler (i.e. Webpack) as part of the ember-cli pipeline and an optional config that allows you to turn on each of the "static flags" of embroider one-by-one.

In the "Inversion of Control" version of the blueprint we intend to keep the same `compatBuild()` API but the job of the builder will be very different. Instead of Vite running as part of the ember-cli pipeline we are only running a prebuild that collects information about your application and its addons to make that available to the Embroider Vite plugin. When running directly in Vite the builder argument to `compatBuild()` will be inert and will do nothing.

To continue to support commands like `ember build` or `ember test` we need some way for ember-cli to interact with Vite and allow Vite to build the application and run tests against the built output. Running `ember test` will essentially run Vite once (as a "one shot build" with no watching functionality) using the builder imported from `@embroider/vite` and then run `ember test --path {outdir}` where `{outdir}` will target the build output from Vite. This allows us to continue to support testem and any CI process that people have defined to use `ember build` without needing to update. Here is an example of an updated `ember-cli-build.js` file:

```diff
 const EmberApp = require('ember-cli/lib/broccoli/ember-app');
+const { compatBuild } = require('@embroider/compat');
+const { builder } = require('@embroider/vite');

 module.exports = function (defaults) {
   let app = new EmberApp(defaults, {});
-  return app.toTree();
+  return compatBuild(app, builder, { /* optional Embroider options */ });
 };
```

The other change that will happen in the next Embroider major release (and will be true for the new blueprint) is that the options passed to `compatBuild()` will have **all the static flags turned on by default**. 

Also some flags, like `staticEmberSource`, `staticAddonTrees`, and  `staticAddonTestSuportTrees` are forced to be on and will throw an error if you try to set them to false. This error will give guidance wherever possible and link to relevant documentation.

### Explicit Bundler Config -> vite.config.mjs

If you are using the current stable release of Embroider then Embroider is generating a Webpack config for you automatically. It is possible for you to make some changes via config but the majority of the Webpack config file is hidden from you. 

This RFC proposes that we don't hide the bundler config any more, we will instead have a standard Vite config file that configures the required Embroider plugins. 

Here is the current version of the proposed vite config: 

```js
import { defineConfig } from 'vite';
import { extensions, classicEmberSupport, ember } from '@embroider/vite';
import { babel } from '@rollup/plugin-babel';

export default defineConfig({
  plugins: [
    classicEmberSupport(),
    ember(),
    // extra plugins here
    babel({
      babelHelpers: 'runtime',
      extensions,
    }),
  ],
});
```

This config is defining 2 "compound plugins" that contain all the functionality needed for an Ember app to be built with Vite. We have split them into `classicEmberSupport()` and `ember()` to communicate that some of the plugins could be considered optional if you aren't using classic Ember features e.g. you have converted all your templates to GJS files.

### Application metadata -> package.json 

In the [v2 Addon format RFC](https://rfcs.emberjs.com/id/0507-embroider-v2-package-format) we introduced the fact that the package.json `ember-addon` MetaData object should be versioned to identify which addons have been upgraded to the new spec. We will be reusing that concept for v2 apps by requiring the following section to be added to the package.json

```json
{
    "ember-addon" {
        "type": "app",
        "version": 2
    }
}
```

This will opt Embroider into modern resolving rules so that it can interoperate properly with bundlers.

### Exports -> package.json

[Package Exports](https://webpack.js.org/guides/package-exports/) is an addition to the ESM resolving rules that all modern bundlers support. It allows you to configure paths that should be importable from ouside your package, as well as giving you a standard way to "redirect" imports that target your package. 

We already use these semantics in Ember applications when we expect `import SomeComponent from 'app-name/components/some-component'` to actually import the file path `/path/to/app-name/app/components/some-component.js`. In this example you can see that the `/app/` subpath has been added to the location. In ember-cli this semantic is handled in broccoli by actually rewriting paths, but this is one of the main reasons why tooling gets confused by our import paths because it's not using a standard method to define our import paths.

The new blueprint will add the following exports section to the package.json by default: 

```json
{
    "./tests/*": "./tests/*",
    "./*": "./app/*"
}
```

This essentially "redirects" all requests to modules in the `app` folder, with the exepction of any path that has `app-name/tests/` at the start. This means that importing your test helpers like `import superHelper from 'app-name/tests/helpers/super-helper'` won't try to import from the path `/path/to/app-name/app/tests/helpers/super-helper.js`

## How we teach this

All of the guides will need to be updated to make sure that we reference the build system correctly. We will also need to make sure that the system we use that automatically builds the tutorial for us can work with the new build system and blueprint.

It will also probably be worthwhile getting Embroider API documentation added to https://api.emberjs.com/ as one of the listed projects on the left-hand side.

The majority of this RFC is written from the perspective of someone that is running `ember new` for the first time on a brand new app, but we will need to make sure to write both upgrade guides and appropriate codemods for anyone that is wanting to upgrade their apps from the old blueprints to this new default. We also need to put some consideration into the experience of people using [ember-cli-update](https://github.com/ember-cli/ember-cli-update) to upgrade Ember versions when we make this the new default. 

It's also important to note that this RFC does not represent the new bluerpint for a Polaris application. This is just upgrading our build system to use more modern and standard tools. Any communication around this RFC change should be explicit that this is a **part** of what is needed for Polaris but this blueprint update is not giving you all of polaris. This is a single step in that direction.

## Drawbacks

### Upgradability

If developers are using ember-cli-update to upgrade their apps there might be a case where in an upcoming version of Ember they will be "opted in" to an Embroider build with Vite. We don't expect this process to be automatic but we do think that we can get most applications across this line with the use of tooling and codemods. One way to mitigate this problem is that we could maintain a "classic blueprint" until the next Ember major release that people could switch to while they are figuring out how to upgrade to Embroider and Vite.

### Webpack support

The blueprint will default to using Vite as a bundler, and we plan to document the process to add support for more bundlers as part of the implementation of this RFC. We had originally intended to provide support for Webpack as a bundler to help people who have already upgraded to the current stable Embroider version and customised their Webpack builds but Webpack support is not trivial to implement. The Ember Tooling Team believes that the benefits of having Vite as the default build experience are so great that we should not delay the implementation of this RFC while we try to backport the Inversion of control implementation of Embroider to Webpack. 

## Unresolved questions

### package.json meta key

The way that Embroider is currently implemented the Ember MetaData in package.json is set with the key `ember-addon` even for applications. On the one hand it seems good that the applications and addons use the same key for this, but on the other hand it may be confusing that the key for the metadata has the word `addon` in it. We could move both addons and apps to just use the metadata key `ember` but that could create chrun with very little benefit.
