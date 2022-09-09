---
stage: recommended
start-date: 2021-10-21T00:00:00.000Z
release-date: 2022-03-21T00:00:00.000Z
release-versions:
  ember-cli: v4.3.0

teams:
  - cli
prs:
  accepted: https://github.com/emberjs/rfcs/pull/772
project-link:
---

# Deprecate Bower Support

## Summary

This RFC proposes to deprecate the following Bower APIs:

- [Blueprint::addBowerPackageToProject](https://ember-cli.com/api/classes/blueprint#method_addBowerPackageToProject)
- [Blueprint::addBowerPackagesToProject](https://ember-cli.com/api/classes/blueprint#method_addBowerPackagesToProject)
- [Project::bowerDependencies](https://ember-cli.com/api/classes/project#method_bowerDependencies) (not public, but used in several addons)
- [Project::bowerDirectory](https://github.com/ember-cli/ember-cli/blob/master/lib/models/project.js#L127) (not public, but used in several addons)

This RFC also proposes to deprecate building Bower packages, coming from
`/bower_components` by default.

## Motivation

While [Bower](https://bower.io/) is still maintained, it's recommended to use an
alternative package manager instead ([npm](https://www.npmjs.com/), [Yarn](https://yarnpkg.com/), [pnpm](https://pnpm.io/), ...). This is also [the official recommendation](https://github.com/bower/bower/blob/master/README.md?plain=1#L7) of the Bower team. Additionally,
Ember CLI stopped emitting a `bower.json` file since [v2.13.0](https://github.com/ember-cli/ember-cli/blob/master/CHANGELOG.md#2130-beta1) and the Ember Guides recommend to
not use Bower anymore as well in the [Compiling Assets](https://guides.emberjs.com/release/addons-and-dependencies/#toc_compiling-assets) section. Lastly, most addons have
dropped support for Bower as well.

Putting this all together, deprecating (and later on removing) Bower support
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

| Key     | Value                                                |
| ------- | ---------------------------------------------------- |
| `for`   | `'ember-cli'`                                        |
| `id`    | `'ember-cli.blueprint.add-bower-package-to-project'` |
| `since` | `{ available: '4.X.X', enabled: '4.X.X' }`           |
| `until` | `'5.0.0'`                                            |

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

| Key     | Value                                                 |
| ------- | ----------------------------------------------------- |
| `for`   | `'ember-cli'`                                         |
| `id`    | `'ember-cli.blueprint.add-bower-packages-to-project'` |
| `since` | `{ available: '4.X.X', enabled: '4.X.X' }`            |
| `until` | `'5.0.0'`                                             |

### `Project::bowerDependencies`

Returns the Bower dependencies (including the development dependencies) for the
project.

Addons that use this method, mostly do so to check the presence of a
specific Bower package in order to suggest users to use the npm equivalent
instead.

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
| `id`    | `'ember-cli.project.bower-dependencies'`   |
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
| `id`    | `'ember-cli.project.bower-directory'`      |
| `since` | `{ available: '4.X.X', enabled: '4.X.X' }` |
| `until` | `'5.0.0'`                                  |

### Building Bower Packages

If Ember CLI detects that the project contains a Bower directory, it will try to
build a Broccoli tree with Bower packages.

When a Bower directory is detected, the following deprecation message will be
triggered:

```
Building Bower packages has been deprecated. You have Bower packages in
`<bower-directory>`.
```

The deprecation message should be triggered when the [`packageBower`](https://github.com/ember-cli/ember-cli/blob/e747ac4c21a8e4a37e158a226dfa8ac048541b1f/lib/broccoli/default-packager.js#L630)
method is called.

**Deprecation details:**

| Key     | Value                                      |
| ------- | ------------------------------------------ |
| `for`   | `'ember-cli'`                              |
| `id`    | `'ember-cli.building-bower-packages'`      |
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

### Building Bower Packages

Building Bower packages has been deprecated.

Please consider one of the following alternatives:

1. Install the package via the npm registry and use `ember-auto-import` to
import the package into your project
2. If alternative 1 is not an option, you could copy the contents of the Bower
package into the `/vendor` folder and use `app.import` to import the package
into your project

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
- Projects that still import Bower packages will need to use one of the
suggested alternatives in the [Building Bower Packages](#building-bower-packages-1)
deprecation guide

## Alternatives

- Keep supporting Bower indefinitely

## Unresolved questions

- Should a deprecation warning be displayed when the `--skip-bower` CLI flag is
used?
