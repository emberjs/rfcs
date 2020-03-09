- Start Date: 2020-03-02
- Relevant Team(s): Ember CLI
- RFC PR: [#599](https://github.com/emberjs/rfcs/pull/599)
- Tracking: (leave this empty)

# Co-located Tests

## Summary

As apps grow, and need to be refactored, the ability to have tightly-coupled concerns grouped together enables faster refactors and better overall upkeep as related files are grouped together -- as has been demonstrated by co-location of Component class/template, the Pods layout, and Module Unification. This RFC proposes to co-locate tests with their concerned behavior under test (component, controller, service, etc), so that drag-and-drop refactoring is easier.

## Motivation

Similar to why people choose the Pods layout, or Module Unification (back when it was available in the canary build), and more recently, component class/template co-location, the ability to group tightly coupled things together allows for much quicker implementation, iteration, and maintenance of any unit of work.

In today's project structures,

 - specific tooling is required to be able to quickly jump to a related test (acceptance tests excluded). By co-locating tests, we eliminate the need for tooling-specific functionality.
 - if you were to mass-migrate a number of files via automation, it is less complex to move and interact with sibling files than it is to try to find the related files in some other top-level directory.
 - it is not clear where a test is located, or if it even exists. Tests for components, for example, can live anywhere in `tests/`, and finding them could be non-trivial for teams that are mid-migration to/from pods or have less than strict PR reviews. Co-locating gives a clear and quick message to the developer that a something has a test.

This aligns with some of the philosophy of the [Module Unification](https://github.com/emberjs/rfcs/blob/master/text/0143-module-unification.md) RFC:

> Subtrees should be relocatable. If you move a directory to a new place in the tree, its internal structure should all still work.

Which becomes more apparent when each subtree makes use of sibling / descendant imports.

## Detailed design

Nothing about existing tests needs to change. All tests currently living in the `tests/` directory may remain there and coexist with co-located tests.

Primary changes needed:

When building an ember app, based on if tests are included in the build or not, as they would be for development and test builds, ember-cli must configure broccoli to skip over test files within the `app` directory.

Additionally, for the test bundle, the `app` directory must be added to search for files that are tests.

Test files could remain using the current hyphenated suffix that they currently use -- example: `my-component-test.js`.
But, because the only name part that could be searched for is `-test` at the end of the filename, there is the possibility that valid components, controllers, routes, etc could legitimately end with `-test`, causing files to incorrectly get removed or added to the app or test bundles.
To counter this conflict, co-located tests must end with `.test`, e.g.: `my-component.test.js`. Because having extra periods in the filename isn't allowed in framework objects, using `.test` should be fairly safe to search for when building the app and test bundles.

More concisely:

For the app bundle,
 - previously
   - include: `app/**/*.{js,hbs}`
 - now
   - include: `app/**/*.{js,hbs}`
   - exclude: `app/**/*.test.{js,ts}`

For the test bundle,
 - previously
   - include: `tests/**/*.{js}`
 - now
   - include: `tests/**/*.{js}`
   - include: `app/**/*.test.{js}`


Like with component co-location, there will be different preferences with how to group files together. It may be undesirable to have a flat list of 20 routes, and then an additional flat list of 20 tests for those routes all in the same folder.
The `.ember-cli` file provides the option to further nest components' class and template files as `app/components/{component-name}/index.{js,hbs}` via the `componentStructure` property (set to `nested`).
This same flexibility should be available for routes, controllers, etc -- but is out of scope for this RFC as other resolver implementations may be in-flight.
Avoiding implementing `componentStructure` for the rest of the `app` directory does not have any impact on test co-location as the resolver is not involved with finding test file locations. For example, it would be valid to have `app/routes/{route-name}.js` and tests located at `app/routes/__tests__/{route-name}.test.js`.

### Addons, in-repo Addons and Engines

For normal addons, living as their own entire project, the lookup rules for apps will apply.
For in-repo addons and engines, there is the possibility that the implementation of this feature would enable things not possible today.
For example, if the "test finder" followed the glob: `**/*(\.test.js|-test.js)`, (excluding node\_modules, etc), that would include tests that are in in-repo addons and engines,
which today, do not support their own tests -- the in-repo addon or engine still wouldn't support their own tests, exactly, as they would run as part of the parent
app's test suite. This might be desired behavior as it would allow the in-repo addon and engine to finally co-locate their tests with the behavior that those units implement.


### Reference: Where do the existing tests get migrated to?
While tests in `app/` could live anywhere, these are to be the default locations for the generators to use.

#### Route

previously:
 - `app/routes/{route-name}.js`
 - `tests/unit/{route-name}-test.js`
now:
 - `app/routes/{route-name}.js`
 - `app/routes/{route-name}.test.js`

#### Service

previously:
 - `app/services/{service-name}.js`
 - `tests/unit/{service-name}-test.js`
now:
 - `app/service/{service-name}.js`
 - `app/service/{service-name}.test.js`

#### Controller

previously:
 - `app/controllers/{controller-name}.js`
 - `tests/unit/{controller-name}-test.js`
now:
 - `app/controllers/{controller-name}.js`
 - `app/controllers/{controller-name}.test.js`

#### Component

previously:
 - `app/components/{component-name}.js`
 - `app/components/{component-name}.hbs`
 - `tests/integration/components/{component-name}-test.js`
now:
 - `app/components/{component-name}.js`
 - `app/components/{component-name}.hbs`
 - `app/components/{component-name}.test.js`

or using `"componentStructure": "nested"`:
 - `app/components/{component-name}/index.js`
 - `app/components/{component-name}/index.hbs`
 - `app/components/{component-name}/index.test.js`

#### Helpers

previously:
 - `app/helpers/{helper-name}.js`
 - `tests/integration/helpers/{helper-name}-test.js`
now:
 - `app/helpers/{helper-name}.js`
 - `app/helpers/{helper-name}.test.js`

#### Utils

previously:
 - `app/utils/{util-name}.js`
 - `tests/unit/{util-name}-test.js`
now:
 - `app/utils/{util-name}.js`
 - `app/utils/{util-name}.test.js`

#### Acceptance Tests

Acceptance tests do not have a core affiliation with any other file, so co-locating them _may_ not make sense, depending on what a particular test file is doing.

While tests may live anywhere in the `app` directory, the default location for acceptance tests will still be `tests/acceptance/`.


## How we teach this


> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

"Test co-location" makes sense, because it's what's used by components' current organization.

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

The guides would need to update any references to unit/integration tests in the `tests` folder to be in the `app` folder.


> How should this feature be introduced and taught to existing Ember
users?

This RFC does not deprecate the old test location -- it only suggests adding test locations -- so people can take their time migrating or completely avoid migrating altogether and wait for a tool to automatically move (nearly) everything for them.

## Drawbacks

While the community migrates, there will be a number of apps that either haven't migrated, or will be in the middle of migrating for a while (depending on app size).
It would be benificial to provide a CLI migration tool to automatically move files from today's standard locations to the new locations.

When moving test files, it's possible that the import paths will be wrong. For example, if someone had `../../app/utils/my-utils.js` in their test file, the path would be wrong if the file moved to a new location.

## Alternatives

The alternatie is to rely on tooling, and implement more features into the (Unstable) Ember Language Server. This will always have gaps, as there are many editors and IDEs out there, and not everyone uses the same editor.

Prior Art:
 - Jest projects -- where co-location happens everywhere
 - Module Unification -- every was co-locatable, even tests -- though there were bugs wit the tests showing up in the app bundle.


## Unresolved questions

- Should the root of test resolution be the project root, or app/addon folder?
  - if project root,
    - this gives ultimate flexibility, and people could literally do whatever they want.
    - but we'd need to also filter out directories, like tmp, dist, node\_modules
  - if app/addon folders,
    - then we'd need a more static list of supported folders for test lookup
      - app, addon, lib, not sure what else
- Given that in-repo addons can't have their own tests, can engines?
- Would bundling all tests in an app into a single test suite have undesirable consequences?
