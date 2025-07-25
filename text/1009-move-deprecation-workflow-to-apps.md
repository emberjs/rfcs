---
stage: released
start-date: 2024-02-22T00:00:00.000Z
release-date:
release-versions:
  ember-cli: 6.3.0
teams:
  - cli
  - data
  - framework
  - learning
  - typescript
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1009'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/1048'
  released: 'https://github.com/emberjs/rfcs/pull/1086'
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

# Move the deprecation workflow library to be installed in apps by default 

## Summary

Historically, folks have benefitted from [ember-cli-deprecation-workflow](https://github.com/mixonic/ember-cli-deprecation-workflow). This behavior is _so useful_, that it should be built in to folks applications by default.

## Motivation

Everyone needs a deprecation-workflow, and yet `ember-cli-deprecation-workflow` is not part of the default blueprint. 

This RFC proposes how we can ship deprecation workflow handling behavior in apps by default, which may give us a blessed path for better integrating with build time deprecations as well (though that is not the focus of this RFC).


## Detailed design


Have `ember-cli-deprecation-workflow` installed by default.

1. applications must have `@embroider/macros` installed by default.
2. the app.js or app.ts can conditionally import a file which sets up the deprecation workflow 
    ```diff app/app.js
      import Application from '@ember/application';
    + import { importSync, isDevelopingApp, macroCondition } from '@embroider/macros';

      import loadInitializers from 'ember-load-initializers';
      import Resolver from 'ember-resolver';
      import config from 'test-app/config/environment';

    + if (macroCondition(isDevelopingApp())) {
    +   importSync('./deprecation-workflow');
    + }

      export default class App extends Application {
        modulePrefix = config.modulePrefix;
        podModulePrefix = config.podModulePrefix;
        Resolver = Resolver;
      }

      loadInitializers(App, config.modulePrefix);
    ```
3. then in `app/deprecation-workflow.js` would use the already public API, 
    ```js
    import setupDeprecationWorkflow from 'ember-cli-deprecation-workflow';

    setupDeprecationWorkflow({
    /**
        false by default, but if a developer / team wants to be more aggressive about being proactive with
        handling their deprecations, this should be set to "true"
    */
      throwOnUnhandled: false,
      handlers: [
        /* ... handlers ... */
      ]
    });
    ```


This follows the README of [ember-cli-deprecation-workflow](https://github.com/ember-cli/ember-cli-deprecation-workflow?tab=readme-ov-file#getting-started).

    

## How we teach this

We'd want to add a new section in the guides under [`Application Concerns`](https://guides.emberjs.com/release/applications/) that talks about deprecations, how and how to work through those deprecations.

All of this content already exists using a similar strategy as above, here, [under "Configuring Ember"](https://guides.emberjs.com/release/configuring-ember/handling-deprecations/#toc_deprecation-workflow), and also walks through how to use `ember-cli-deprecation-workflow`. 

This README of [ember-cli-deprecation-workflow](https://github.com/ember-cli/ember-cli-deprecation-workflow?tab=readme-ov-file#getting-started) also explains, in detail, how to use this tool / workflow, and that content can be copied in to the guides.

## Drawbacks

For older projects, this could be _a_ migration. But as it is additional blueprint boilerplate, it is optional.

## Alternatives

There are only a few features of `ember-cli-deprecation-workflow` that we need to worry about:
- enabled or not - do we check deprecations at all, or ignore everything (current default)
- `throwOnUnhandled` - this is the most aggressive way to stay on top of your deprecations, but can be frustrating for folks who may not be willing to fix things in `node_modules` when new deprecations are introduced.
  
- `window.flushDeprecations()` - prints the list of deprecations encountered since the last page refresh
- Matchers - a fuzzier way to match deprecation messages rather than strictly matching on the deprecation id (sometimes deprecation messages have information about surrounding / relevant context, and these could be used to more fine-grainedly work through large-in-numbers deprecations)
- Logging / Ignoring / Throwing - when encountering a matched deprecation (whether by id or by regex, how should it be handled?)


However, folks can get a basic deprecation-handling workflow going in their apps without the above features,

1. applications must have `@embroider/macros` installed by default.
2. the app.js or app.ts can conditionally import a file which sets up the deprecation workflow 
    ```diff app/app.js
      import Application from '@ember/application';
    + import { importSync, isDevelopingApp, macroCondition } from '@embroider/macros';

      import loadInitializers from 'ember-load-initializers';
      import Resolver from 'ember-resolver';
      import config from 'test-app/config/environment';

    + if (macroCondition(isDevelopingApp())) {
    +   importSync('./deprecation-workflow');
    + }

      export default class App extends Application {
        modulePrefix = config.modulePrefix;
        podModulePrefix = config.podModulePrefix;
        Resolver = Resolver;
      }

      loadInitializers(App, config.modulePrefix);
    ```
    this conditional import is now easily customizable for folks in their apps, so they could opt to _not_ strip deprecation messages in production, and see where deprecated code is being hit by users (reported via Sentry, BugSnag, or some other reporting tool) -- which may be handy for folks who have a less-than-perfect test suite (tests being the only current way to automatically detect where deprecated code lives).
3. the `app/deprecation-workflow.js` would use the already public API, [`registerDeprecationHandler`](https://api.emberjs.com/ember/5.6/functions/@ember%2Fdebug/registerDeprecationHandler)
    ```ts
    import { registerDeprecationHandler } from '@ember/debug';

    import config from '<app-moduleName>/config/environment';

    const SHOULD_THROW = config.environment !== 'production';
    const SILENCED_DEPRECATIONS: string[] = [
      // Add ids of deprecations you temporarily want to silence here.
    ];

    registerDeprecationHandler((message, options, next) => {
      if (!options) {
        console.error('Missing options');
        throw new Error(message);
      }

      if (SILENCED_DEPRECATIONS.includes(options.id)) {
        return;
      } else if (SHOULD_THROW) {
        throw new Error(message);
      }

      next(message, options);
    });
    ```


This simple implementation of deprecation workflow may work for libraries' test-apps, but it is not as robust as what `ember-cli-deprecation-workflow` offers, per the above-listed set of features that folks are used to.

To get all of those features from `ember-cli-deprecation-workflow`, we could define a function, `setupDeprecationWorkflow`, taken from the [Modernization PR on ember-cli-deprecation-workflow](https://github.com/mixonic/ember-cli-deprecation-workflow/pull/159), this is what the deprecation-workflow file could look like:

<details><summary>ember-cli-deprecation-workflow/index.js</summary>

```js
import { registerDeprecationHandler } from '@ember/debug';

const LOG_LIMIT = 100;

export default function setupDeprecationWorkflow(config) {
  self.deprecationWorkflow = self.deprecationWorkflow || {};
  self.deprecationWorkflow.deprecationLog = {
    messages: {},
  };

  registerDeprecationHandler((message, options, next) =>
    handleDeprecationWorkflow(config, message, options, next),
  );

  registerDeprecationHandler(deprecationCollector);

  self.deprecationWorkflow.flushDeprecations = flushDeprecations;
}

let preamble = `import setupDeprecationWorkflow from 'ember-cli-deprecation-workflow';

setupDeprecationWorkflow({
  workflow: [
`;

let postamble = `  ]
});`;

export function detectWorkflow(config, message, options) {
  if (!config || !config.workflow) {
    return;
  }

  let i, workflow, matcher, idMatcher;
  for (i = 0; i < config.workflow.length; i++) {
    workflow = config.workflow[i];
    matcher = workflow.matchMessage;
    idMatcher = workflow.matchId;

    if (typeof idMatcher === 'string' && options && idMatcher === options.id) {
      return workflow;
    } else if (typeof matcher === 'string' && matcher === message) {
      return workflow;
    } else if (matcher instanceof RegExp && matcher.exec(message)) {
      return workflow;
    }
  }
}

export function flushDeprecations() {
  let messages = self.deprecationWorkflow.deprecationLog.messages;
  let logs = [];

  for (let message in messages) {
    logs.push(messages[message]);
  }

  let deprecations = logs.join(',\n') + '\n';

  return preamble + deprecations + postamble;
}

export function handleDeprecationWorkflow(config, message, options, next) {
  let matchingWorkflow = detectWorkflow(config, message, options);
  if (!matchingWorkflow) {
    if (config && config.throwOnUnhandled) {
      throw new Error(message);
    } else {
      next(message, options);
    }
  } else {
    switch (matchingWorkflow.handler) {
      case 'silence':
        // no-op
        break;
      case 'log': {
        let key = (options && options.id) || message;

        if (!self.deprecationWorkflow.logCounts) {
          self.deprecationWorkflow.logCounts = {};
        }

        let count = self.deprecationWorkflow.logCounts[key] || 0;
        self.deprecationWorkflow.logCounts[key] = ++count;

        if (count <= LOG_LIMIT) {
          console.warn('DEPRECATION: ' + message);
          if (count === LOG_LIMIT) {
            console.warn(
              'To avoid console overflow, this deprecation will not be logged any more in this run.',
            );
          }
        }

        break;
      }
      case 'throw':
        throw new Error(message);
      default:
        next(message, options);
        break;
    }
  }
}

export function deprecationCollector(message, options, next) {
  let key = (options && options.id) || message;
  let matchKey = options && key === options.id ? 'matchId' : 'matchMessage';

  self.deprecationWorkflow.deprecationLog.messages[key] =
    '    { handler: "silence", ' + matchKey + ': ' + JSON.stringify(key) + ' }';

  next(message, options);
}
```

</details>

and at this point, we may as well build it into `ember` and not use an additional library at all, **and this is what the primary proposal of this RFC: build the deprecation workflow setup function in to ember**, so re-running through the setup steps:

1. applications must have `@embroider/macros` installed by default.
2. the app.js or app.ts can conditionally import a file which sets up the deprecation workflow 
    ```diff app/app.js
      import Application from '@ember/application';
    + import { importSync, isDevelopingApp, macroCondition } from '@embroider/macros';

      import loadInitializers from 'ember-load-initializers';
      import Resolver from 'ember-resolver';
      import config from 'test-app/config/environment';

    + if (macroCondition(isDevelopingApp())) {
    +   importSync('<app-moduleName>/deprecation-workflow');
    + }

      export default class App extends Application {
        modulePrefix = config.modulePrefix;
        podModulePrefix = config.podModulePrefix;
        Resolver = Resolver;
      }

      loadInitializers(App, config.modulePrefix);
    ```
    this conditional import is now easily customizable for folks in their apps, so they could opt to _not_ strip deprecation messages in production, and see where deprecated code is being hit by users (reported via Sentry, BugSnag, or some other reporting tool) -- which may be handy for folks who have a less-than-perfect test suite (tests being the only current way to automatically detect where deprecated code lives).
3. the `app/deprecation-workflow.js` would use the already public API, [`registerDeprecationHandler`](https://api.emberjs.com/ember/5.6/functions/@ember%2Fdebug/registerDeprecationHandler)
    ```js
    import { setupDeprecationWorkflow } from '@ember/debug';

    setupDeprecationWorkflow({
      throwOnUnhandled: true,
      handlers: [
        /* ... handlers ... */
      ]
    });
    ```



## Unresolved questions

n/a
