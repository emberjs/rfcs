- Start Date: (2015-06-16)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Instead of accessing functions and classes directly on the `Ember` namespace,
move all the public API under ES6 modules.

# Motivation

As build tooling improved during the recent years, especially with Ember CLI, it
is now fairly straightforward to rely only on ES6 modules. This comes with a
number of benefits:

1. Future-proofing. Since modules are part of the ECMAScript specification, it
   is clear that they become commonplace fairly soon. Ember has long been
   pushing the approach, this RFC just continues what's been started a couple
   years ago with Ember AppKit, followed by Ember CLI.
2. Makes possible to distribute ember in smaller packages. This could further
   slim down Ember's infamous byte size - but note that this may not mean
   completely separate NPM modules. Some libraries are even worthy of
   distributed by themselves, for example the module-based resolver is already a
   separate package. Some notable others are also simply external libraries
   packaged alongside Ember.
3. Treeshaking. Since dependencies could be determined at compile-time based on
   the `import` statements, parts of Ember that are not used by an application
   would not end up in the final build. Think of including only
   `HistoryLocation` when an app doesn't need to support hashchange-only
   browsers.

# Detailed design

## End-user syntax

Take the following Ember CLI example:

```js
import Ember from 'ember';

export Ember.Component.extend({
  someMethod(arg) {
    return Ember.isBlank(arg);
  }
});
```

Here, both `Ember.Component` and `Ember.isBlank` could be `import`ed:

```js
import Component from 'ember-component';
import { isBlank } from 'ember-utils';

export Component.extend({
  someMethod(arg) {
    return isBlank(arg);
  }
});
```

It worth noting that many people already declaring variables for the above
imports, simply because it just feels right to treat Ember's parts as modules,
even if the `import` syntax is not available today except for `Ember`:

```js
import Ember from 'ember';
const { Component, isBlank } = Ember;
```

## List of modules

The complete list of modules has been put together by the core team, and is
available under the following URL: https://docs.google.com/spreadsheets/d/1g312T6-5kFizpaL8J3Dqx3iOyloDoBdWF3tZMT6xHCs/edit#gid=0

**Note that all modules not present in Ember 2.0 are dropped.** The shims for
the pre-2.0 modules should be distributed in addons alongside with the code that
maintains the backwards compatibility.

## Migration path

Probably the easiest part of the migration to real ES6 modules is extending
[ember-cli-shims](https://github.com/ember-cli/ember-cli-shims) [(canary
branch)](https://github.com/ember-cli/ember-cli-shims/blob/canary/app-shims.js)
with the above module names. Syntax-wise this allows an app developer to use
`import` for all the proposed modules, as soon as it ends up in Ember CLI - even
if Ember remains exactly the same under the hood. Releasing it early would allow
developers to begin the migration to ES6 modules until the time Ember itself
offer the same functionality without shims. This would include updating the
[blueprints](https://github.com/ember-cli/ember-cli/tree/master/blueprints) as
well, to make newly generated files use the new syntax.

For existing files, a migration utility is to be created that converts all
occurrences of the `Ember.*` syntax to their `import` equivalents, possibly in
[ember-watson](https://github.com/abuiles/ember-watson).

Next, Ember itself is to be updated to export these modules, and move everything
else (internal usage) into a clearly private module path.

As the final step, publish Ember directly to NPM as an addon with the new module
API. This would sunset the distribution of Ember using Bower, and would make
`ember-cli-shims` obsolete.


# Drawbacks

One of the drawbacks is losing the ease of use of services like JSBin. Even if
[DefaultResolver](http://emberjs.com/api/classes/Ember.DefaultResolver.html) is
being kept, the following is unlikely to be tolerable for an end-user:

```js
App.XFooComponent = require('ember-component')['default'].extend();
```

The other issue is, if Ember is going to be distributed as an NPM package,
consisting of ES6 modules only, the above is not even possible easily - at least
not without compiling and distributing Ember on a CDN.

# Alternatives

# Unresolved questions

For how long should the current globals be maintained? Possibly Ember 3.0?
