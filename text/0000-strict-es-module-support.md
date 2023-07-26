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
  accepted: # update this to the PR that you propose your RFC in
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

## Transition Path

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

- For cases where enumerating modules is truly necessary and legitimate, we're offering up a companion RFC to this one introducing `import.meta.glob`. The biggest difference between `import.meta.glob` and `requirejs.entries` is that you can only `import.meta.glob` your own package's files. If an addon wants to enumerate files from the app, you need to ask the app author to pass you the `import.meta.glob()` results. See [the RFC](#fixme) for details.


## How We Teach This

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?
Does it mean we need to put effort into highlighting the replacement
functionality more? What should we do about documentation, in the guides
related to this feature?
How should this deprecation be introduced and explained to existing Ember
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.
There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
