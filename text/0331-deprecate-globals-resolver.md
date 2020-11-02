---
Start Date: 2018-05-08
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/331
Tracking: https://github.com/emberjs/rfc-tracking/issues/20

---

# Summary

Deprecate all use of:

- Ember Globals Resolver (looks up a class via a global namespace such as "App")
- Creation of a Global Namespace (`var App = Ember.Namespace.create();`)
- Ember.TEMPLATES array
- &lt;script type="text/handlebars" data-template-name="path/to/template"&gt;

Use of any of the above should trigger a deprecation warning, with a target
of version 4.0

# Motivation

Over the past years we have transitioned to using Ember-CLI as the main way
to compile Ember apps. The globals resolver is a holdover and primarily
facilitates use of Ember without Ember-CLI.

# The Globals Resolver

For those who are not aware, the globals resolver is available via `@ember/globals-resolver` or
`Ember.DefaultResolver`. For more information, see the
[api](https://www.emberjs.com/api/ember/release/classes/GlobalsResolver/properties).
Using it looks like the following:

```js
// app.js
var App = Ember.Application.create();

App.Router.map(function() {
  this.route('about');
});

App.AboutRoute = Ember.Route.extend({
  model: function() {
    return ['red', 'yellow', 'blue'];
  }
});
```

```html
// index.html
<script type="text/x-handlebars" data-template-name="about">
  <ul>
    {{#each model as |item|}}
      <li>{{item}}</li>
    {{/each}}
  </ul>
</script>
```

# Implementation Details

One small detail required to implement this RFC: ember-cli's own default resolver,
[ember-resolver](https://github.com/ember-cli/ember-resolver)
currently still extends from the globals resolver.
In order to implement this RFC, the ember-cli resolver will need to be changed
so that it does *not* extend from the globals resolver, or otherwise ember-cli users
will get a deprecation warning as well.
However, changing the base class of the ember cli classic resolver is a breaking change,
so prior to ember/ember-cli version 4.0 we need to take another step.
In the ember-cli classic resolver, deprecate any runtime calls where there is fallback to the globals mode resolver. This would be a deprecation in ember-cli's resolver. We could bump a major version of ember-cli-resolver removing the base class and release it in ember-cli after an LTS of ember-cli.

# Transition Path

Primarily, the transition path is to recommend using Ember-CLI.

During the 3.x timeframe, it MAY become increasingly difficult to use this old functionality.
For example, with the release of 3.0, we already stopped publishing builds that support
globals mode. Here are some of the changes that have impacted or may soon impact users of globals mode:

## Impact of ES6 modules

Users of ES6 modules must use their own build tooling to convert them to named AMD modules via Babel.
No support is provided for &lt;script type="module"&gt; at this time, although that may change.

## Impact of New Module Imports

Globals based apps are only able to use new module imports via the polyfill available at
https://github.com/ember-cli/babel-plugin-ember-modules-api-polyfill No build support for this is provided.

## Impact of not publishing globals builds

It is necessary to get a globals build of Ember.js from the npm package now that globals builds
are no longer published to S3, builds.emberjs.com, and CDNs.

## Impact of not Generating a Globals Build in Ember.js Package

At some point during the 3.x cycle, it may be that we no longer publish a globals build in the
npm package. At that point, it may become necessary to use Ember-CLI to generate a globals build
of Ember.js

## Impact of Package Splitting

Work has started on package splitting. It is likely that the globals resolver may not be included
in a default partial build of Ember.js and may be moved to its own package for easy removal.

## Impact of Tree Shaking

If the globals resolver is moved to a separate package, it will likely not be included in a build
of Ember.js by default unless tree shaking is turned off.

# How We Teach This

We already do teach this and don't teach the globals resolver. No changes required here.

## Deprecation Guide

A draft deprecation guide has been pull requested at https://github.com/ember-learn/deprecation-app/pull/155

# Drawbacks

A drawback is that people may want alternate build tooling to Ember-CLI.
We have mitigated this by openly publishing the ember-cli resolver and all parts of the
ember-cli ecosystem under the MIT license.
Alternate build tooling may simply use this open source code to build a competing
infrastructure to ember-cli.

# Alternatives

Without doing this, we will have to continue to ship and maintain this rarely used functionality.
We don't believe this is a reasonable alternative.

# Unresolved questions

There has never been a transition guide for transitioning an old codebase to Ember-CLI.
Do we want to create one at this late date?
