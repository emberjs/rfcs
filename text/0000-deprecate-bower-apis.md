---
Stage: Accepted
Start Date: 2021-10-21
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember CLI
RFC PR:
---

# Deprecate Bower APIs

## Summary

This RFC proposes to deprecate the following Bower APIs:

- [Blueprint::addBowerPackageToProject](https://ember-cli.com/api/classes/blueprint#method_addBowerPackageToProject)
- [Blueprint::addBowerPackagesToProject](https://ember-cli.com/api/classes/blueprint#method_addBowerPackagesToProject)
- [Project::bowerDependencies](https://ember-cli.com/api/classes/project#method_bowerDependencies) (not public, but used in several addons)
- [Project::bowerDirectory](https://github.com/ember-cli/ember-cli/blob/master/lib/models/project.js#L127) (not public, but used in several addons)

## Motivation

While [Bower](https://bower.io/) is still maintained, it's recommended to use an
alternative package manager instead ([npm](https://www.npmjs.com/), [Yarn](https://yarnpkg.com/), [pnpm](https://pnpm.io/), ...). This is also [the official recommendation](https://github.com/bower/bower/blob/master/README.md?plain=1#L7) of the Bower team. Additionally,
Ember CLI stopped emitting a `bower.json` file since [v2.13.0](https://github.com/ember-cli/ember-cli/blob/master/CHANGELOG.md#2130-beta1) and the Ember Guides recommend to
not use Bower anymore as well in the [Compiling Assets](https://guides.emberjs.com/release/addons-and-dependencies/#toc_compiling-assets) section. Lastly, most addons have
dropped support for Bower as well.

Putting this all together, deprecating (and later on removing) all Bower APIs
seems like the best path forward.

## Transition Path

### `Blueprint::addBowerPackageToProject`

Adds a Bower package to the project's `bower.json` file.

When used, a deprecation message will be triggered suggesting to use
[Blueprint::addPackageToProject](https://ember-cli.com/api/classes/blueprint#method_addPackageToProject) instead:

```
`addBowerPackageToProject` has been deprecated. If the package is also available
on the npm registry, please use `addPackageToProject` instead. If not, please
suggest your users to install the Bower package manually by running:
`bower install <package-name> --save`.
```

**Deprecation details:**

| Key     | Value                                            |
| ------- | ------------------------------------------------ |
| `for`   | `'ember-cli'`                                    |
| `id`    | `'ember-cli.blueprint.addBowerPackageToProject'` |
| `since` | `{ available: '4.X.X', enabled: '4.X.X' }`       |
| `until` | `'5.0.0'`                                        |

### `Blueprint::addBowerPackagesToProject`

Adds multiple Bower packages to the project's `bower.json` file.

When used, a deprecation message will be triggered suggesting to use
[Blueprint::addPackagesToProject](https://ember-cli.com/api/classes/blueprint#method_addPackagesToProject) instead:

```
`addBowerPackagesToProject` has been deprecated. If the packages are also available
on the npm registry, please use `addPackagesToProject` instead. If not, please
suggest your users to install the Bower packages manually by running:
`bower install <package-name> <package-name> --save`.
```

**Deprecation details:**

| Key     | Value                                             |
| ------- | ------------------------------------------------- |
| `for`   | `'ember-cli'`                                     |
| `id`    | `'ember-cli.blueprint.addBowerPackagesToProject'` |
| `since` | `{ available: '4.X.X', enabled: '4.X.X' }`        |
| `until` | `'5.0.0'`                                         |

### `Project::bowerDependencies`

Returns the Bower dependencies (including the development dependencies) for the
project.

Addons that use this method, mostly do so to check the presence of a
specific Bower package in order to suggest users to use the npm equivalent instead.

When used, the following deprecation message will be triggered:

```
`bowerDependencies` has been deprecated. If you still need access to the
project's Bower dependencies, you will have to manually resolve the project's
`bower.json` file instead.
```

**Deprecation details:**

| Key     | Value                                      |
| ------- | ------------------------------------------ |
| `for`   | `'ember-cli'`                              |
| `id`    | `'ember-cli.project.bowerDependencies'`    |
| `since` | `{ available: '4.X.X', enabled: '4.X.X' }` |
| `until` | `'5.0.0'`                                  |

### `Project::bowerDirectory`

Returns the project's Bower directory.

When used, the following deprecation message will be triggered:

```
`bowerDirectory` has been deprecated. If you still need access to the
project's Bower directory, you will have to manually resolve the project's
`.bowerrc` file and read the `directory` property instead.
```

**Deprecation details:**

| Key     | Value                                      |
| ------- | ------------------------------------------ |
| `for`   | `'ember-cli'`                              |
| `id`    | `'ember-cli.project.bowerDirectory'`       |
| `since` | `{ available: '4.X.X', enabled: '4.X.X' }` |
| `until` | `'5.0.0'`                                  |

### Additional Tasks

- Remove all Bower references from the `app` and `addon` blueprints

## Deprecation Guide

### `Blueprint::addBowerPackageToProject`

`addBowerPackageToProject` has been deprecated. If the package is also available
on the npm registry, please use `addPackageToProject` instead. If not, please
suggest your users to install the Bower package manually by running:

```
bower install <package-name> --save
```

### `Blueprint::addBowerPackagesToProject`

`addBowerPackagesToProject` has been deprecated. If the packages are also available
on the npm registry, please use `addPackagesToProject` instead. If not, please
suggest your users to install the Bower packages manually by running:

```
bower install <package-name> <package-name> --save
```

### `Project::bowerDependencies`

`bowerDependencies` has been deprecated. If you still need access to the
project's Bower dependencies, you will have to manually resolve the project's
`bower.json` file instead:

```js
'use strict';

const fs = require('fs-extra');
const path = require('path');

module.exports = {
  name: require('./package').name,

  included() {
    this._super.included.apply(this, arguments);

    let bowerPath = path.join(this.project.root, 'bower.json');
    let bowerJson = fs.existsSync(bowerPath) ? require(bowerPath) : {};
    let bowerDependencies = {
      ...bowerJson.dependencies,
      ...bowerJson.devDependencies,
    };

    // Do something with `bowerDependencies`.
  },
};
```

### `Project::bowerDirectory`

`bowerDirectory` has been deprecated. If you still need access to the
project's Bower directory, you will have to manually resolve the project's
`.bowerrc` file and read the `directory` property instead:

```js
'use strict';

const fs = require('fs-extra');
const path = require('path');

module.exports = {
  name: require('./package').name,

  included() {
    this._super.included.apply(this, arguments);

    let bowerConfigPath = path.join(this.project.root, '.bowerrc');
    let bowerConfigJson = fs.existsSync(bowerConfigPath) ? fs.readJsonSync(bowerConfigPath) : {};
    let bowerDirectory = bowerConfigJson.directory || 'bower_components';

    // Do something with `bowerDirectory`.
  },
};
```

## How We Teach This

The following references to Bower in the Ember Guides should be removed:

- [Compiling Assets](https://guides.emberjs.com/release/addons-and-dependencies/#toc_compiling-assets)

The following references to Bower in the Ember CLI Guides should be removed:

- [Where Should Assets and Dependencies Go?](https://cli.emberjs.com/release/basic-use/assets-and-dependencies/#whereshouldassetsanddependenciesgo)
- [Assets and Dependencies](https://cli.emberjs.com/release/advanced-use/debugging/#assetsanddependencies)
- [ember addon](https://cli.emberjs.com/release/advanced-use/cli-commands-reference/#emberaddon)
- [ember init](https://cli.emberjs.com/release/advanced-use/cli-commands-reference/#emberinit)
- [ember new](https://cli.emberjs.com/release/advanced-use/cli-commands-reference/#embernew)
- [Blueprint Hooks](https://cli.emberjs.com/release/writing-addons/addon-blueprints/#blueprinthooks)
- [WebStorm](https://cli.emberjs.com/release/appendix/dev-tools/#webstorm)

We should also make sure to remove all references to Bower in the
[Ember CLI API documentation](https://ember-cli.com/api/).

_NOTE: It's possible that some references are not included in the lists above._

## Drawbacks

- Addons that still rely on a Bower package which does not yet have an npm
equivalent would need to suggest their users to manually install said package
- Addons that still require access to the project's Bower dependencies, will
have to manually resolve the project's `bower.json` file instead
- Addons that still require access to the project's Bower directory, will
have to manually resolve the project's `.bowerrc` file and read the `directory`
property instead

## Alternatives

- Keep supporting Bower indefinitely

## Unresolved questions

- Should a deprecation warning be displayed when the `--skip-bower` CLI flag is
used?
- Is deprecating these Bower APIs sufficient to completely remove Bower support
in the future? If not, what is still missing in this RFC?
