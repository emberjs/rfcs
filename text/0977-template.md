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
building off the prior work in [v2 Addon format](https://github.com/emberjs/rfcs/pull/507),
that is designed to make Ember apps more compatible with the rest of the JavaScript ecosystem. This RFC will define conventions of the app such that it is _possible_ to confidently build an ember app without ember-cli (e.g.: in jsbin). 

## Motivation

Ember has a long-standing tradition of pioneering developer-friendly conventions, allowing developers to build powerful web applications with minimal configuration. Over the years, we have embraced the philosophy of "convention over configuration," making it easy for developers to get started and achieve consistency across Ember projects.

However, as the JavaScript ecosystem evolves, it becomes increasingly important for Ember to adapt and interoperate seamlessly with the broader web development landscape. The introduction of a v2 app format is driven by several key motivations:

1. Compatibility with the JavaScript Ecosystem

Ember's success lies not only in its robust features but also in its ability to integrate effortlessly with other JavaScript libraries and tools. With the v2 app format, we aim to make Ember applications more compatible with the rest of the JavaScript ecosystem. This means that developers should feel confident in building Ember applications without the need for specialized tooling like ember-cli. Whether it's experimenting with code in an environment like jsbin or integrating Ember into projects that use native packagers like Vite, Webpack, or Turbopack, the v2 app format should empower developers to work seamlessly with Ember in diverse contexts.


2. Improved Insight into App Initialization

Ember's boot process lacks clear, documented specifications. This lack of clarity means that developers often struggle to comprehend the precise steps involved in booting an Ember app manually (if they ever attempt to do so).

The v2 app format aims to rectify this by providing comprehensive documentation that outlines the intricacies of an Ember app's initialization process. By shedding light on the sequence of steps necessary to bootstrap an Ember application, developers will gain a deeper understanding of the framework's internal mechanisms. This newfound knowledge will empower developers to confidently control and customize the app initialization process, enabling them to tailor Ember apps to their specific needs.

3. Performance and Optimization

Performance is a critical concern for modern web applications. The v2 app format is designed to deliver tangible benefits to Ember developers and users. Most importantly, all capabilities of a chosen packager (be that Vite, Webpack, turbopack, etc) would become available to Ember (module federation, alternative SSR methods, etc).

4. Interoperability with Native Packagers

Embracing native packagers such as Vite, Webpack, and Turbopack is crucial for staying current with JavaScript ecosystem trends. The v2 app format is designed to accommodate these packagers seamlessly, ensuring that Ember applications can harness the benefits of these tools while maintaining compatibility.

5. Future-Proofing

Lastly, the v2 app format positions Ember to take advantage of ongoing and future developments in the wider JavaScript ecosystem related to code bundling and optimization. By aligning with industry trends, Ember remains a forward-looking framework, ready to embrace new technologies and practices.

## Detailed design

How much of this should be "the path/migration to" as opposed to "this is the end state, details can be part of implementation"?

1. start with embroider-strictest
2. all blueprint addons must be either in the v2 format, or non-addons entirely (such as qunit-dom's recent change to real type=module package) 
3. use Vite w/ embroider
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

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
