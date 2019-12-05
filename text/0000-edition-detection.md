- Start Date: (2019-11-20)
- Relevant Team(s): (Ember.js, Ember CLI, Ember Data)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Edition detection

## Summary

Introduces a mechanism that an application can use to specify which Edition of
Ember it is intending to target. This RFC will define:

* How to specify the edition that the application is using
* How other packages (addons, codemods, etc) can detect the applications intended edition
* What the edition should be used for

## Motivation

As Ember approaches it's first edition (see
[emberjs/rfcs#364](https://github.com/emberjs/rfcs/pull/364)) various addons
need to modify their behavior based on the edition that is being used. An
initial implementation (done without RFC) used the `setEdition` method from
`@ember/edition-utils` inside the application's (or addon's) `.ember-cli.js`
file to specify which edition was to be used. That implementation worked well
enough throughout the intial preview period, but a number of major issues were
(rightfully!) surfaced by the community:

1. it seems unnecessary/redundant
2. it's not clear what this flag actually does (likely due to having no RFC!)
3. it's not statically analyzable (and therefore cannot be used by things like codemods)

This RFC will review these issues in light of the updated implementation,
showing how each of the concerns have been met.

## Detailed design

### Specifying the edition

A new entry will be added to the project's `package.json`:

```js
{
  "name": "project-name",
  // ...snip
  "ember": {
    "edition": "octane"
  }
}
```

The value will be expected to be one of the valid edition names (currently
`classic` and `octane`). Using the `package.json` for this allows us to ensure
that the value is statically analyzable and easy to discover.

### Valid use of the edition value

The edition flag should only be used by addons to determine what blueprint
output to generate and to provide helpful warnings (or errors) at build time.

### Detecting the edition

The existing `@ember/edition-utils` package will still be used by addons to
detect which edition is in use, but it will be updated to check the new
location (instead of relying on folks leveraging `setEdition`).

The API documentation for `@ember/edition-utils` would be:

```ts
module '@ember/edition-utils' {
  /**
    Determine if the application that is running is running under a given Ember
    Edition.  When the edition in use is _newer_ than the requested edition it
    will return `true`, and if the edition in use by the application is _older_
    than the requested edition it will return `false`.

    @param {string} requestedEditionName the Edition name that the application/addon is requesting
    @param {string} [projectRoot=process.cwd()] the base directory of the project
  */
  has(requestedEditionName: string, projectRoot?: string): boolean;

  /**
    Sets the Edition that the application should be considered a part of.
    This method is deprecated, and will be phased out in the next major release.

    @deprecated
  */
  setEdition(editionName: string): void;
}
```

For a period of time the `@ember/edition-utils` package will continue to
support existing users of `setEdition` when an edition is not detected via the
new mechanism. This allows users that have been testing out Ember Octane
(either via the `@ember/octane-app-blueprint` or manually using `setEdition` in
their `.ember-cli.js`) a period of time in order to migrate.

## How we teach this

The official guides at `https://guides.emberjs.com/release/configuring-ember/`
will be updated to include documentation of the new `pacakge.json`
configuration and clearly explain what the edition flag is used for (warnings
and blueprints).

This will will not be a difficult concept to teach to folks (most users won't
care, and will get upgrade as part of a future `ember-cli` blueprint update).

## Drawbacks

> Changing existing app and addon usage of the prior flag will cause churn.

This is significantly mitigated by ensuring that `@ember/edition-utils`
continues to support users of `setEdition` API as a fallback (with a
deprecation), and that the existing `has` API continues to work (defaulting the
project root to the current working directory).

## Alternatives

### Use `emberEdition` in `package.json`

Some folks may prefer to use a single new property in `package.json` (vs the
`"ember": { "edition": "octane" }` setup proposed above). I personally think it
makes more sense to start with an `"ember":` key, as there are additional
possible usages (e.g. moving `"ember-addon"` configuration to be within
`"ember"`) and migrating from `emberEdition` to the nested syntax would be
needless churn.

### Use `.ember-cli` instead of `package.json`

Instead of adding the `emberEdition` value to the `package.json` we could add
it to the existing `.ember-cli` file. However, doing this would **not** satisfy
the static analysis constraint mentioned in the motivation section (because
`.ember-cli.js` is transparently supported by `ember-cli`'s build system).

## Unresolved questions

TBD?
