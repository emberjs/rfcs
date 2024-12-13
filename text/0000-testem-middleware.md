---
stage: accepted
start-date: 2023-12-06
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - learning
prs:
  accepted: https://github.com/emberjs/rfcs/pull/993
project-link:
suite: 
---

# Let the app define Testem middleware in testem.js rather than implicitly import middleware only from v1 addons

## Summary

Ember CLI should let Ember apps customize Testem middleware. An Ember app should be able to provide its own middleware, while also being able to opt in or out of middleware provided by v1 and v2 addons.

## Motivation

Testem middleware is a way for Ember addons to customize the Express server that [Testem](https://github.com/testem/testem/tree/f2bc5cc1cf0b63b4e8cee992d3e16317bfeef4c3) starts to serve an Ember app for testing. Such middleware is provided for various purposes by [a handful](https://emberobserver.com/code-search?codeQuery=testemMiddleware) of Ember addons, including ones of infrastructural importance: `ember-cli-code-coverage`, `ember-cli-typescript`, `ember-cli-fastboot-testing`.

Current Ember CLI implementation ([code 1](https://github.com/ember-cli/ember-cli/blob/v5.4.1/lib/tasks/test.js#L30-L42), [code 2](https://github.com/ember-cli/ember-cli/blob/v5.4.1/lib/tasks/test.js#L59), [API docs](https://ember-cli.com/api/classes/addon#method_testemMiddleware)) only collects the `testemMiddleware` hooks defined by addons and then uses it to override `middleware` defined in the `testem.js` config file.

Only v1 addons are able to define Ember CLI hooks. v2 addons are expected to instruct the consumer to configure their app by hand. This implies that an Ember app should be able to import middleware from v2 addons and pass it to Ember CLI. The lack of this functionality blocks some v1 addons from migrating to the v2 format.

Ideally, an Ember app should be able to do three things:

1. Define its own middleware.
2. Pass on middleware provided by v2 addons.
3. Opt in or out of Ember CLI including middelware provided by v1 addons.

## Detailed design

There is already a nice convention we should build on top of: an Ember app [has](https://github.com/ember-cli/ember-new-output/blob/v5.4.1/testem.js) a `testem.js` config file at root level. This file is Testem's [own config](https://github.com/testem/testem/blob/f2bc5cc1cf0b63b4e8cee992d3e16317bfeef4c3/docs/config_file.md?plain=1#L90).

Unfortunately, Ember CLI [overrides](https://github.com/ember-cli/ember-cli/blob/v5.4.1/lib/tasks/test.js#L59) the `middleware` option of the Testem config with middleware [collected from v1 addons](https://github.com/ember-cli/ember-cli/blob/v5.4.1/lib/tasks/test.js#L30-L42).

Instead, **Ember CLI should check whether `middleware` is defined in the app's `testem.js`**:

* If `middleware` is not present in `testem.js`, fall back to the [original behavior](https://github.com/ember-cli/ember-cli/blob/v5.4.1/lib/tasks/test.js#L59) of providing middleware collected from v1 addons via `this.addonMiddlewares(options)`.
* If it's present, then Ember CLI should do nothing, letting Testem consume middleware defined in the app.

✨ The latter moves the responsibility of managing Testem middleware from Ember CLI blackbox magic into the userland of an Ember app. ✨

### Usage examples

The `middleware` option in `testem.js` is an [optional array](https://github.com/testem/testem/blob/f2bc5cc1cf0b63b4e8cee992d3e16317bfeef4c3/docs/config_file.md?plain=1#L90). In this array, a maintainer of an Ember app can:

1. Define their own middleware functions, e. g.:

    ```js
    // testem.js

    module.exports = {
      // ...
      middleware: [
        (expressAppInstance) => { /* ... */ },
        (expressAppInstance) => { /* ... */ },
      ],
    };
    ```

2. Pass middleware imported from v2 addons, e. g.:

    ```js
    // testem.js

    import middlewareImportant from 'ember-important-addon';
    import middlewareFancy from 'ember-fancy-addon';

    module.exports = {
      // ...
      middleware: [
        middlewareImportant,
        middlewareFancy,
      ],
    };
    ```

3. Opt into importing middleware from v1 addons.

    v1 addons define Testem middleware as a `testemMiddleware` hook in each addon's `index.js`. Those hooks are not exported and are not consumable by `testem.js`.

    Thus, we need to provide an util that would collect middleware hooks from all v1 addons and return them as an array.

    ```js
    // testem.js

    import collectTestemMiddlewareFromV1Addons from 'ember-cli';

    module.exports = {
      // ...
      middleware: [
        ...collectTestemMiddlewareFromV1Addons(),
      ],
    };
    ```

4. A combination of the above.

### The implementation of collectTestemMiddlewareFromV1Addons

The `collectTestemMiddlewareFromV1Addons` util should do exactly what `addonMiddlewares` [does](https://github.com/ember-cli/ember-cli/blob/v5.4.1/lib/tasks/test.js#L30-L42): collect the `testemMiddleware` hooks from v1 addons and return them as an array of curried functions.

There are a number of unsolved questions around it:

* ❓ How should it be implemented?

    The challenge is that `addonMiddlewares` is a method on the `TestTask` class of Ember CLI, whereas `collectTestemMiddlewareFromV1Addons` is supposed to be a simple function executed by Testem, outside of Ember CLI context. In order to read the `testemMiddleware` hooks from all addons consumed by the app, the util probably needs to instantiate a subset of Ember CLI.

* ❓ Should the name `collectTestemMiddlewareFromV1Addons` be changed to something else?

* ❓ Where should `collectTestemMiddlewareFromV1Addons` be imported from?

* ❓ Which file should its impementation code reside in the Ember CLI codebase?

## How we teach this

* For **addon maintainers**, we should: 
  * Update the [Writing addons](https://cli.emberjs.com/release/writing-addons/) guide, obliging addon maintainer to update their readme files and release/upgrade notes with instructions that tell the user to manipulate the `testem.js` config by hand.
  * The inline API documentation for modified Ember CLI hooks and methods will be updated.
  * A scheduled blog post with update information will be posted, announcing the change.
* For **addon consumers**, there is nothing to be done, as Ember CLI behavior does not change for them until they decide to upgrade an addon from v2. In that case, they will read release notes with instructions.

❓ With this approach, it's the addon maintainers' responsibility to provide information about `collectTestemMiddlewareFromV1Addons`. If an addon fails to mention that, the user may be in trouble, as v1 addons that rely on Testem middleware will break.

## Drawbacks

An "Ember app" mentioned throughout this RFC means a v1 app, as v2 apps (Embroider-first apps) do not exist yet. v2 apps will not be driven by Ember CLI, and, given the alternative described below, this work will become redundant.

❓ Something else?

## Alternatives

### Use the in-repo addon workaround

The fact that some important Ember CLI hooks are only available to addons and not the app — is a known problem in the Ember community. A de-facto solution is to add an in-repo addon to the app. Such addon can define a middleware hook that provides both inline middleware and forwards middleware from v2 addons.

Pros:

* Does not require any changes in Ember CLI.
* Works today.

Cons:

* This pattern is not eagerly documented and only known to seasoned Ember developers.
* This pattern is a workaround for an Ember CLI shortcoming, not a real solution.
* Relying on Testem's own config file is the v2 path forward, and using an in-repo addon delays it.

### Have Ember CLI merge middleware from addons with middleware from testem.js

Pros:

* Likely a one-liner that can be categorized as a bugfix.

Cons:

* It solves only two goals out of three:

    ✅ Define its own middleware.  
    ✅ Pass on middleware provided by v2 addons.  
    ❌ Opt in or out of Ember CLI including middelware provided by v1 addons.

    Must admit that opting out of addon-provided Testem middleware is a rare requirement.

* It does not push app and addon maintainers to adopt the Embroider-friendly path.

## Unresolved questions

❓ Should this feature be backported to older Ember CLI versions?