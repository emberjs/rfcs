---
stage: accepted
start-date: 2024-11-30T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: 
  - cli
  - framework
  - learning
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1054
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

# external default blueprints in ember-cli 

## Summary

External blueprints give us the ability to rapidly iterate and distribute changes to the blueprint. They allow us to release changes at a pace that is more frequent than the release train. This decoupling is highly desirable, and doesn't diminish the purpose and goals of the release train process. 

## Motivation

Since we're wanting to introduce new blueprints, for both the [v2 app][gh-v2-app-blueprint] and [v2 addon][gh-v2-addon-blueprint], as default, we need to quickly iterate and release updates. The release-train that ember-cli abides by is useful to protect users from any instability that may occur during active development. However, since blueprints are always opt in -- either via new project, or updating an old project, it is safe to for blueprints to have separate release cadences from the core ember-cli project. 


[gh-v2-app-blueprint]: https://github.com/embroider-build/app-blueprint
[gh-v2-addon-blueprint]: https://github.com/embroider-build/addon-blueprint
[rfc-v2-app-blueprint]: https://github.com/emberjs/rfcs/pull/977
[rfc-v2-addon-blueprint]: https://github.com/emberjs/rfcs/pull/985 

## Detailed design

**tl;dr, behavior changes**
1. external blueprints are added to `dependencies` of `ember-cli`. Default behavior uses the resolved path from `dependencies`
2. ember-cli will prompt the user if they wish to use the latest version of the blueprint if one is available 
3. ember-cli always prints which blueprint it is using and from where
4. ember-cli will not print warnings on custom flags used in blueprints

**tl;dr: for external blueprints**
1. same node support as ember-cli 
2. must (also) test against windows
3. if the blueprint generates a project, every package.json#script must be tested and exit with status 0 (success).

**tl;dr, non-behavior changes**
1. this RFC does not change any of the current blueprints (other RFCs may, (such as for the [v2 app][rfc-v2-app-blueprint] and the [v2 addon][rfc-v2-addon-blueprint]))
2. this RFC does not suggest we change anything about how ember-cli releases


The main thing to retain with external blueprints is the output of a blueprint must be the same for all of time when a version of ember-cli is released. This is, in part, to help the upgrade story for folks batching many versions of upgrades at once, so they have roughly the same experience as folks who upgrade their apps as updates are released.

To solve this, ember-cli would want to use specific versions of blueprints for each release. These can be managed via `dependencies` and without a range character (`~`, `^`, etc). The `--blueprint` argument already accepts a path, and we can configure `dirname(require.resolve('blueprint-name')` to pass to `--blueprint`.

While pinning would ensure long-term repeatability, it does mean that we accidentally lose one of the benefits of external blueprints -- frequent updates. With the release-train that ember-cli uses, with no further changes -- and including for all blueprints prior -- users must wait **¼ year**, best case scenario, from time of PR to release of ember-cli, before blueprint updates may be adopted via using the stable release of ember-cli.  

To resolve, ember-cli will query, if there is an internet connection, what the latest version of the external blueprint is, and if it is newer than the current bundled (defined in `dependencies`) version: the user will be prompted if they want to use the bundled version of the blueprint or the latest version. 

For [example][example-ember-clack-external-blueprints]:
```
┌   ember new 
│
◆  There is a newer version of `@embroider/app-blueprint` available. Would you like to use it?
│  ● 2.0.1 (default)
│  ○ 3.4.19 (latest)
└
```

Since there is a possibility of ember-cli using different versions, we need to show which version and at what path is of the blueprint.

For [example][example-ember-clack-external-blueprints]:
```
┌   ember new 
│
●  Using blueprint from /home/projects/clack-prompts/node_modules/@embroider/app-blueprint
│
```
or 
``` 
┌   ember new 
│
◇  There is a newer version of `@embroider/app-blueprint` available. Would you like to use it?
│  3.4.19 (latest)
│
●  Using blueprint from /tmp/abc123/hash/@embroider/app-blueprint
│
│
◇  Installed via npm
│
```

The exact details of the prompts here fall under implementation detail, and are not prescribed by this RFC.

[example-ember-clack-external-blueprints]: https://stackblitz.com/edit/clack-prompts-gg5p7c?file=package.json

### Upgrade paths 

In order for upgrades to be smooth for external blueprints, blueprints need to be able to "feature detect" the situations, environment, and capabilities of the project that an upgrade is being applied to. For the existing blueprint system, it means that ember-cli will need to allow blueprints to have any custom flags they blueprint deems necessary. For example, in the `@embroider/app-blueprint`, it will want to detect, upon upgrade, if `hbs` support is needed, and may represent that as `--hbs-support=true|false`, which would become available via the blueprint `options` object passed around to all the blueprint function hooks. So feature detection needs to occur _before_ the options object is built. This would be an addition to ember-cli's blueprint system, we can call it `detectFeatures()`, which returns a Promise which resolves to an object, which would be spread in to the `options` object, allowing the features to be configured in the blueprint's `locals`, which allows the blueprint-diff infrastructure to appropriately upgrade the app. 

Example:

```js 
// my-blueprint/index.js (CommonJS)
'use strict';

const { getFeatures } = require('ember-cli/lib/blueprint-utils');
const path = require('path');
const globSync = require('....');


module.exports = {
    /**
     * Called during `init` (used when upgrading)
     * Skipped during `new` and `addon` (nothing to detect)
     *
     * @param {string} targetDirectory the directory that the blueprint is running in (the app to upgrade)
     */
    async detectFeatures(targetDirectory) {
        return {
            hbs: globSync('**/*.hbs', { cwd: targetDirectory }).length > 0,
            typescript: pathExists(path.join(targetDirectory, 'tsconfig.json')), 
        }
    },

    locals(options) {
        let features = getFeatures(options) ?? { 
            /* default for new / addon */ 
            hbs: true, 
            typescript: false,
        };

        return {
            name,
            /* ... */
            features,
        }
    }
}
```

and then in the `vite.config.mjs`:
```js 
import {
  // ...
<% if (features.hbs) { %>
    hbs,
<% } %>
  // ...
} from '@embroider/vite';

// ...
export default defineConfig(({ mode }) => {
    return {
        plugins: [
        <% if (features.hbs) { %>
          hbs(),
        <% } %>
        }
        // ...
    }
```

### Prerequisites for an external blueprint to become default 

Note that these do not, nor should not, precede the planning / RFC work for proposing an external blueprint to become default (as to not block future or presently in-progress RFCs).

An External Blueprint needs to:
1. have its CI run against the same node support range as ember-cli.
2. be able to run all its tests against Windows  
3. Adhere to SemVer
4. If the external blueprint generates a project, there should be a set of "slow" / acceptance tests that assure that all package.json#scripts can succeed.

## How we teach this

The terminal output and guidance should be enough for most users.

For blueprint authors, this RFC defines the prerequisites for using an external blueprint as default in ember-cli. 

## Drawbacks

- more code in ember-cli
- no release-train for blueprints

## Alternatives

- do nothing (likely continued friction between ember-cli releases and blueprint development)
- unpinned blueprints in ember-cli releases (or `~` ranges in `dependencies`)
- no interactive prompt to use latest (would make it harder for users to know there may be potential bugfixes)

## Unresolved questions

none yet
