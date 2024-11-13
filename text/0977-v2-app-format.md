---
stage: accepted
start-date: 2023-10-06
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
  - learning
prs:
  accepted: https://github.com/emberjs/rfcs/pull/977 
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

Over the past year the Ember Core Tooling team have been working hard to invert the control between bundlers and ember-cli, which means that instead of ember-cli running a bundler (such as Webpack) as part of its build system the whole Ember build process will essentially become a plugin to the bundler. This means that we can more effectively make use of bundler innovations, performance improvements, and we are more capable of adapting to whatever next generation of build systems come to the Javascript ecosystem.

With the Ember build system as a plugin to bundlers (such as Vite or Webpack) we also have the ability to only intervene on things that are Emberisims (i.e. not-standard) and as we work to make Ember more standard we can eventually turn off these ”compatibility plugins”. Compatibility plugins will need to be powered by an “ember prebuild” that collects information about your app and addons and output metadata that the bundler plugins will consume. The intention is that this prebuild will be done automatically for you as part of the bundler plugin setup and you should not have to worry about preforming extra steps.

This RFC is not going to describe a new blueprint where you don't need any compatibility plugins (or an ember prebuild) to run an Ember app, this RFC instead is going to propose a new blueprint that has all the compatibility plugins turned on so that it is easiest for most people to upgrade to. Any discussion about a minimal “compatibility-free” blueprint should happen in a later RFC. 

## Key Ideas

Much like the [v2 Addon format RFC](https://rfcs.emberjs.com/id/0507-embroider-v2-package-format#key-ideas) we want the new app blueprint to rely on ES Modules and not leave anything hidden or automatic. This not only makes it easier for developers to know where things are coming from, it also makes it easier for bundlers to know what to do with Ember applications.

Each of the following sections go into detail of all the changes between the current blueprint and the new proposal which is currently being developed at https://github.com/embroider-build/app-blueprint

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

In a classic build ember-cli would collect all the of the modules in your app into an entrypoint in the build that would go through and define each of the modules in the require.js AMD registry. There is already an RFC that describes the fact that we want to deprecate this AMD registry, but the key thing for this RFC is that we are trying to think of our app as series of real ES Modules we need to provide some way for the built-in discovery of these modules that allows Ember to still resolve those modules by name (for Dependency Injection concerns like services)

We are providing this with the virtual module `'@embroider/virtual/compat-modules';`. This means that any bundler plugin that wants to support Ember needs to be able to support virtual module imports. The contents of this file can be obtained by asking the Embroider resolver which collates the list of modules during the Ember prebuild.

We then pass the list of modules to an updated version of the Ember Resolver (that has already been released )

TODO add an example and keep describing

TODO add a comment about eagerness with the compat module import

### Application Config -> app/config/environment.js and config/environment.js

### Test Entrypoint -> tests/index.html and tests/test-helper.js

### Explicit Babel Config -> babel.config.cjs

### Ember Pre-Build Config -> ember-cli-build.js

### Explicit Bundler Config -> vite.config.mjs



How much of this should be "the path/migration to" as opposed to "this is the end state, details can be part of implementation"?

1. start with embroider-strictest
2. all blueprint addons must be either in the v2 format, or non-addons entirely (such as qunit-dom's recent change to real `type=module` package) 
3. use Vite with Embroider
4. remove unneeded dependencies
    - ember-auto-import
        - replaced by _the packager's_ own way of processing dependencies
            (requires all consumed dependencies to _not_ be v1 addons)
    - broccoli-asset-rev
        - replaced by _the packager's_ production build mode
    - ember-cli-babel
        - replaced by actual babel 
    - ember-cli-clean-css
        - replaced by _the packager's_ production build mode
    - ember-cli-htmlbars
        - replaced by [babel-plugin-ember-template-compilation](https://github.com/emberjs/babel-plugin-ember-template-compilation/) and embroider-provided build plugins 
    - ember-cli-inject-live-reload
        - replaced by _the packager's_ development mode
    - ember-cli-sri
        - replaced by _the packager's_ production build mode
    - ember-cli-terser
        - replaced by _the packager's_ production build mode
    - ember-fetch
        - requires work in `@ember/test-helpers` and `ember-fetch` to make this smooth.
    - ember-load-initializers
        - tbd
    - ember-resolver
        - tbd
    - loader.js
        - require / AMD has been private for a long time, and folks should not be using it.
    

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

In this section I'll 

## Unresolved questions

### Application config

In the current design of the new build we are still using the node-generated config that all ember developers are used to. We have considered unifying the `config/environemnt.js` and `app/config/environment.js` files to reduce complexity in the learning story but
