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
that is designed to make Ember apps more compatible with the rest of the JavaScript ecosystem. This RFC will define conventions of the app and clearly identify functionality that is considered "compat" and could be considered optional in the near future.

## Motivation

When ember-cli was created there was no existing JS tooling that met the needs of the Ember Framework. Over the years we have added more and more developer-friendly conventions to our build system that many Ember applications and addons depend on. As the JavaScript tooling story has gotten a lot better of the years Ember has fallen behind because our custom-built tools have not been keeping up with the wider community. Efforts have been started to improve the situation with the advent of [Embroider](https://github.com/embroider-build/embroider) but the current stable release of Embroider still runs inside ember-cli and is somewhat bound in performance and capability by the underlying technology [broccolijs](https://github.com/broccolijs/broccoli).

Over the past year the Ember Core Tooling team have been working hard to invert the control between the bundler and ember-cli, which means that instead of ember-cli running the bundler as part of its build system the whole Ember build process will essentially become a plugin to the bundler. This means that we can more effectively make use of bundler innovations, performance improvements, and we are more capable of adapting to whatever next generation of build systems come to the Javascript ecosystemm.

With the Ember build system as a plugin to bundlers (such as Vite or Webpack) we also have the ability to only intervene on things that are Emberisims (i.e. not-standard) and as we work to make Ember more standard we can eventually turn off these compat plugins.

This RFC is not going to describe a new blueprint where you don't need any compatibility plugins to run an Ember app, this RFC instead is going to propose a new blueprint that has all the compatibility plugins turned on so that it is easiest for most people to upgrade to.

## Key Ideas

Much like the [v2 Addon format RFC](https://rfcs.emberjs.com/id/0507-embroider-v2-package-format#key-ideas) we want the new app blueprint to rely on ES Modules and not leave anything hidden or automatic. This not noly makes it easier for developers to know where things are coming from, it also makes it easier for bundlers to know what to do with Ember applications.

Each of the following sections go into detail of all the changes between the current blueprint and the new proposal which is currently being developed at https://github.com/embroider-build/app-blueprint

## Detailed design

In this section I'm going to go through each of the changes in the new proposed blueprint, in each section I will do my best to explain the reasoningin for why it needs to be like this but if you have any questions or comments please feel free to comment on the RFC and we will try to update that section.

### Entrypoint -> index.html

### App Entrypoint -> app/app.js

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
