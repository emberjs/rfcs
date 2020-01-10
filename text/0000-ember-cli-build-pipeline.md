- Start Date: 2020-01-10
- Relevant Team(s): Ember CLI
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking:

# Ability to hook into build process without addons

## Summary

This RFC proposes adding the `treeFor` and `treeFor*` hooks to the options passed to the `EmberApp` class.
These hooks would run during the build process and allow developers to affect the build
pipelines for each of the trees (app, vendor, public, etc).

## Motivation

Currently, the only supported way to affect the Broccoli tree during build time is by hooking into
the `treeFor*` APIs provided by addons. Developers that want to use these APIs are forced to develop
an addon or in-repo addon. This is problematic for a few reasons:

- An addon is a full npm package. Even if it’s an in-repo addon, it comes with its own name and
dependency tree. The burden of maintaining another package to write a few lines
of code perpetuates the perception that Ember is "bloated" and has a steep learning curve.
- Writing code in an addon (or `lib/`) requires relying Ember CLI to invoke the code. There is no
guarantee of when or how the code will be called, so it keeps developers locked into the Ember
ecosystem. This feeling of being trapped in a build system means under-investment,
or worse: moving away.
- The entry point of the Ember build is the `ember-cli-build.js` file. The full power of a good
program should always be available in the entry point of the program. Being able to add modules
into the program should be treated as a progressive enhancement rather than a requirement.
- Although any number of addons can be added into the build process and any number of them could
affect the build process, there is no good way to guarantee the order of these addons. With hooks
exposed in a single place, it would be simple to execute code in the desired order.
- Any framework-agnostic business logic, configuration, or data needed by the build pipeline
must first be deployed independently as a module and then wrapped again in an ember-specific addon. This is a maintenance nightmare.

## Detailed design

The high level of Ember application build looks like this:

```js
const app new EmberApp(defaults, {});
return app.toTree();
```

The `toTree()` method on the Application is responsible for gathering modules from the app
and addons and outputing the `vendor.js` and an `app.js` file. In addition to creating the final bundle,
it also runs the `treeFor` hooks from addons. It would be possible to add functionality to refactor
this into also running `treeFor` hooks on the `EmberApp` instance itself.

## How we teach this

The Ember CLI documentation currently does not introduce Ember CLI as a build pipeline. Rather,
it explains Ember CLI as an ecosystem of addons, and covers common use cases such as asset compilation
and minification. This can make it challenging to dive into the build pipeline, when it is necessary.

Teaching these `treeFor` hooks would require first making the decision to expose the build pipeline
as a first-class citizen of the Ember CLI ecysostem.

## Drawbacks

None.

## Alternatives

Wait for Embroider and see what it offers?

## Unresolved questions

- Would these hooks be run before or after the addon hooks and will that cause any complications?
