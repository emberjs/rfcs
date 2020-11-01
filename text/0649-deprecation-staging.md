---
Start Date: June 18, 2020
Relevant Team(s): Ember.js, Learning, Steering, Ember CLI
RFC PR: https://github.com/emberjs/rfcs/pull/649
Tracking: 

---

# Deprecation Staging

## Summary

Establish a staging system for deprecations that allows Ember and other
libraries to roll them out more incrementally. This system will also enable
users to opt-in to a more strict usage mode, where deprecations will assert if
encountered.

The initial stages would be:

1. Available
2. Enabled

## Motivation

Ember has a robust deprecation system that has served the community well. However,
there are a number of pain points and use cases that it does not cover very well:

* **Ecosystem Absorption**: Generally, deprecations need to be absorbed by the
  addon ecosystem before they can be absorbed by applications. This is because
  apps may see a deprecation that is triggered by addon code, that they have no
  control over (other than jumping in and helping out the addon author by reporting
  an issue or sending in a pull request). However, today there is no way to add a
  deprecation just for
  addons - adding a deprecation will make it appear everywhere at once.

  This creates pressure to only add deprecations when most of the community has
  already transitioned away from the API. However, it also means that we can't
  send a strong signal, like a deprecation, to let the community know that it
  _should_ begin transitioning. This cyclic dependency can make it difficult to
  push the community forward over time.

  Deprecation staging would allow deprecations to be added incrementally.
  Combined with default settings for apps and addons that enable deprecations at
  different stage levels, this would allow incremental rollout throughout the
  ecosystem.

* **Pacing**: Currently, deprecations must have their timeline built
  directly into the deprecation RFC from the beginning. The deprecation API requires an `until` version,
  and there isn't much flexibility during the rollout process because of this.
  Once a deprecation is RFC'd and merged, there isn't much the community can do
  to slow down the process.

  Staging would give the community multiple checkpoints to decide
  whether or not a deprecation was ready to move forward in the process, and to
  update the timeline if so. This would allow a deprecation to slow down or
  speed up based on real world feedback and absorption.

* **Early Adoption**: Since the current system discourages us from adding
  deprecations until we are sure they are ready, it also prevents early adopters
  from getting good signals about which patterns are on the way out. This
  prevents them from exploring replacements, which means the ecosystem has even
  less ability to adapt to deprecations and move forward.

  Staging would allow us to introduce deprecations and allow early adopters to
  explore them. This would allow us to determine if the new APIs that have been
  introduced fully cover the use cases of the APIs that are being deprecated.

* **Restricting Usage/Preventing Backsliding**: Finally, many times deprecations
  are fully removed from an Ember app or addon, only to accidentally be added
  again in the future. Since deprecations only produce a console warning by default,
  it is difficult and unintuitive to prevent this today.

  With staging, users will be able to opt-in to deprecations becoming
  assertions at a particular stage, in a non breaking way. This will allow users
  to prevent backsliding as they incrementally migrate their applications to new
  APIs. It will also provide the basis for future RFCs to explore compiling away deprecated features -
  if they throw an error when used, they can't be used, so they can also be
  removed and the app will continue to work.

## Detailed design

There are two stages that are proposed for deprecations in this RFC:

1. **Available**: Deprecations are initially released in the available stage.
   Available deprecations are enabled by default in addons, but not in apps.
   This is explained in more detail later.

2. **Enabled**: Enabled is the final stage for deprecations. Deprecations
   should be moved into Enabled after the replacement APIs have moved into
   Recommended in the RFC process, and there is generally high confidence that
   the addon ecosystem has absorbed the deprecation for the most part.
   Enabled deprecations cannot be disabled.

Deprecations will progress through these stages linearly. In the future, more
stages could be added, but the progression will remain linear. Deprecations that
involve an RFC (as most do), should lay out a plan for how and when a deprecations
will advance to the next stage, but _this_ RFC does not try to prescribe that
progress.

There are two mechanisms that will need new behavior for the proposed staging
system:

* The `deprecate` function
* The `ember` config on `package.json`

### `deprecate`

The current deprecation function receives the following options:

```ts
interface DeprecationOptions {
  id: string;
  until: string;
  url?: string;
}
```

This will be updated to the following:

```diff
interface DeprecationOptions {
  id: string;
  until: string;
  url?: string;
+ for: string;
+ since: {
+   available: string;
+   enabled: string;
+ }
}
```

Stepping through the new options individually:

* `for`: This is used to indicate the library that the deprecation is for.
  `deprecate` is a public API that can be used by anyone. Now that we intend to
  show or hide deprecations based on their stage, we need to have some more
  information about what library the deprecation belongs to. It should be the
  package name for the library. This can be any string, but in the future, we may
  be able to loosen this requirement by using a macro that can get the name of
  the package that invokes `deprecate()`.

* `since`: This property contains a set of keys corresponding to
  each deprecation stage. If a value for a stage if present, the value of all
  previous stages must also be present. Each key's value is an exact SemVer
  value. To start, only two keys (`available` and `enabled`) will
  be allowed, and the rest will be ignored.

  Any SemVer value can be used here, even if it does not correspond to a published
  artifact in a package registry. This is for the sake of simplicity and flexibility,
  but it should be noted that creating a deprecation that becomes available in the
  future is not recommended.

  The highest stage that has a value is considered the stage of the
  deprecation is in. Any of the values may be used for deprecation compliance
  assertions, depending on the app or addon configuration. This is explained
  more below.

If the `since` key is not provided (as is the case for existing deprecations),
the deprecation is assumed to be `enabled`. This RFC also... wait for it... deprecates
the usage of the `deprecate()` function without a `since` key. This meta deprecation
will be enabled at the same time as it becomes available like this:

```js

// 3.21.0 is fabricated here. It will be the next version that is released
// after this RFC is merged / when the work is done.
const NEXT_VERSION = '3.21.0';

deprecate('Calling deprecate() without the \'since\' key is deprecated', false, {
  id: 'ember-source.deprecate',
  until: '4.0.0',
  since: {
    available: NEXT_VERSION,
    enabled: NEXT_VERSION,
  }
})
```

### Configuring Apps

Users can configure their application using the `ember.deprecations`
property in `package.json`. This property is empty by default, but can be
customized to indicate which versions of an addon the app is compliant with.
This configuration can be described like this:

```ts
type DeprecationStage = 'available' | 'enabled';
type Editions = 'classic' | 'octane';

interface AddonDeprecationConfig {
  stage: DeprecationStage;
  version: StrictSemVerString;
  display?: DeprecationStage;
}

interface EmberPackageJSONKey {
  edition?: Editions;
  deprecations?: {
    display?: DeprecationStage;
    addons?: {
      [packageName: string]?: AddonDeprecationConfig;
    }
  }
}
```

The new `deprecations` key has two keys of its own:

- `display`: which configures the default stage of deprecations the app owner
  wants to see from their app and all addons. If this config is not provided, it defaults to `'enabled'`.
- `addons`: which declares which versions of each library the app is compliant with.

    Each key points to an object conforming to the `AddonDeprecationConfig` interface.

    Note that while deprecations can be created by applications themselves
    (by calling `deprecate()` in app code), the top level key is still called `addons`.
    The reasoning is that _most_ of the time, Ember does not need to know about
    deprecations from applications, as it needs no build-time signaling. These deprecations
    will always be displayed and treated as if they are at the `enabled` stage. Because
    there is no compliance config, they will not be promoted to runtime assertions.

### AddonDeprecationConfig

The keys in each `AddonDeprecationConfig` are:

* `stage`: This declares compliance with the addon's deprecations to the stage given.
* `version`: This declares compliance with the addon's deprecations to the version given.
* `display`: This overrides the root level `display` config, to allow fine grain control over logged deprecations.

The `stage` and `version` keys are explained in more detail below.

### Deprecation Compliance

Users can opt-in to some deprecations becoming assertions. They do this by
stating that their application is _compliant_ with a given library's
deprecations as of a certain stage and version.

This means that the app is guaranteeing that it does not use _any_ deprecated
APIs that were in _the given stage_ in _the given version_.
If they _do_ use an API that was deprecated prior to version specified, the invocation of `deprecate` will
throw an error instead of logging a warning. The error will contain the same
message, with an additional note that the user is encountering this error
because of their deprecation compliance settings.

Deprecation compliance cannot be specified for a version of the
library newer than the one that is installed. Attempting to do this will throw an
error. This is to prevent developers from optimistically declaring compliance with
code that they are not yet running. This also prevents an edge case scenario where
compliance is declared with a version that has not been released yet and accidentally
opting into unknown deprecations when that version _does_ get released.

### Parsing Compliance Declarations

Apps can declare compliance with an addon in two ways:

- by omitting a declaration
- by passing an object conforming to the `AddonDeprecationConfig` interface.

If a declaration is omitted, Ember will assume that the app is compliant with
all deprecations in the "enabled" stage, but will not opt the app into any assertions.
This ensures backwards compatibility with apps that do not add this config.

If a `AddonDeprecationConfig` object is passed, Ember will assume that the app
is compliant with deprecations of the configured `stage` and later, and the
configured `version` and earlier. Deprecations that fit this configuration
will also throw assertions in development builds.

Note that `AddonDeprecationConfig` must have both the
`version` and the `stage` properties to be be valid. If both are not provided
a build time error will be thrown.

### Usage and Config Example

Let's say that we use `deprecate()` to add a new deprecation to our library.
Because it's new, we're going to put it in the "available" stage.

```js
deprecate('this feature is deprecated!', false, {
  id: 'my-awesome-library.my-awesome-feature',
  for: 'my-awesome-library',
  since: {
    available: '1.2.3'
  },
  until: '2.0.0'
});
```

There are a few common variations of compliance config:

- No config

    ```json
    {
      "ember": {}
    }
    ```

    This means that an app may not be compliant with the deprecation from
    `my-awesome-library`. Because Ember doesn't have an explicit declaration,
    it will not promote the deprecation to an assertion, regardless of which
    version of `my-awesome-library` is installed.

- Later stage, current version

    ```json
    {
      "ember": {
        "deprecations": {
          "addons": {
            "my-awesome-library": {
              "stage": "enabled",
              "version": "1.2.3"
            }
          }
        }
      }
    }
    ```

    This indicates that the app is not compliant with `my-awesome-feature`
    deprecation, so Ember will not promote it to an assertion, regardless of which
    version of `my-awesome-library` is installed.

- Current stage, but older version.

  ```json
  {
    "ember": {
      "deprecations": {
        "addons": {
          "my-awesome-library": {
            "stage": "available",
            "version": "1.0.0"
          }
        }
      }
    }
  }
  ```

  This config indicates that the app is not compliant with deprecations added
  after 1.0.0, so Ember will not promote the `my-awesome-feature` deprecation to
  an assertion.

- Matching stage and version

    ```json
    {
      "ember": {
        "deprecations": {
          "addons": {
            "my-awesome-library": {
              "stage": "available",
              "version": "1.2.3"
            }
          }
        }
      }
    }
    ```

    This indicates that the app _is_ compliant with the `my-deprecation-feature` and
    Ember will throw a runtime assertion if this deprecation is invoked.

Now consider that `my-awesome-feature` becomes a enabled deprecation in 1.5.0:

```diff
deprecate('this feature is deprecated!', false, {
  id: 'my-awesome-library.my-awesome-feature',
  for: 'my-awesome-library',
  since: {
    available: '1.2.3',
+   enabled: '1.5.0',
  },
  until: '2.0.0'
});
```
The configurations described above would be affected in the following ways:

- No config: no change
- Later stage, current version: no change, because it was not enabled in 1.2.3
- Current stage, older version: no change
- Matching stage and version: no change

In order to opt-in to an assertion for this enabled config, the app would have
to update their declaration to match the `since.enabled` value:

```json
{
  "ember": {
    "deprecations": {
      "addons": {
        "my-awesome-library": {
          "stage": "enabled",
          "version": "1.5.0"
        }
      }
    }
  }
}
```

### Configuring Addons

Addons are also able to declare their compliance with other libraries in the same
way as above. A declaration from an addon tells Ember that its _parent_ addon or application
cannot declare compliance to a higher version.

This allows _application_ developers to feel confident that if they declare a
compliance with a _high_ version, Ember's tooling will tell them early that they
have incompatible addons installed.

An addon can specify its deprecation config in the same way as an application,
but there are some notable differences:

#### Limited to ember-source

Addons are recommended to declare compliance _only_ with the `ember-source` library.
This recommendation does not _prevent_ this mechanism from being used with other libraries,
but the core team would like to exercise this feature before recommending good
patterns and behavior. A warning will be logged in development, if addons declare
compliance with libraries other than `ember-source`.

#### Declaration and Defaults

While applications must set a `StrictSemVerString` value for
`addons.[packageName].version`, addons can additionally set a special
value of `"auto"`, to indicate that the addon is compliant with
deprecations from the _minimum_ version of the SemVer range specified by the
`dependencies` or `devDependencies` in its `package.json`. The configured stage
would still be used as that stage of declared compliance.

The benefit of these new semantics is that the majority of addon authors don't
have to declare that they are compliant and runtime assertions are not introduced
to apps without any major version updates.

Additionally, this `auto` keyword would allow addon authors to use automation
(e.g Dependabot) to update their package's dependencies and get valuable CI
feedback about their addon violating new deprecations, without having to update
any compliance config.

#### Example

Let's look at a complete example with a real world scenario. In 3.15.0,
Ember deprecated [use of the `isVisible` property][1] in components. Although
Deprecation Stages did not exist at the time, this is how that deprecation would
be created after this RFC:

```js
deprecate('\`isVisble\` is deprecated!', false, {
  id: 'ember-component.is-visible',
  for: 'ember-source', // new property
  until: '4.0.0',
  since: {
    available: '3.15.0',
    enabled: '3.16.0', // this is fabricated for this example
  }
});
```

This means that the deprecation for the `isVisible` property became _available_
in 3.15, but became enabled in 3.16.

Now let's consider an addon that has a component that uses `isVisible`, but
doesn't declare any deprecation compliance. In other words, it does not make
any changes or release any new versions. Its `package.json` config looks like this:

```json
{
  "devDependencies": {
    "ember-source": "~3.16.0"
  },
  "ember": {}
}
```

Because no compliance is declared, Ember will not make any assumptions about
this addon's compliance. On the other hand, Ember will also not actively
_prevent_ an app from declaring its own compliance. This not only lets apps
use this feature without updating every addon in the ecosystem, it also provides
incentive for the community to add compliance declarations into every addon, so
that they can provide valuable signals to apps and Ember.

**TODO**: if the app declares it's compliant with available+3.15 deprecations,
and uses this addon, should Ember promote the deprecation to a runtime assertion?
Pro: apps know immediately that they are not actually compliant when they thought
they were. Con: the only way to "unbreak" an app is to update the addon or to lower
their own compliance config -- both are manual steps.

Next, let's consider that the addon declares its compliance using the `auto` keyword instead.
(All new addons will have this config from the addon blueprint, so this will be the
next most common scenario.)

```json
{
  "devDependencies": {
    "ember-source": "~3.16.0"
  },
  "ember": {
    "deprecations": {
      "addons": {
        "ember-source": {
          "stage": "available",
          "version": "auto"
        }
      }
    }
  }
}
```

Here, the addon is actively declaring that it is compliant with all available
deprecations in `ember-source` 3.16.0, but not in 3.16.1 and beyond. Because this is an active
declaration, Ember will use it for two things:

1. It will not allow parent addons/apps to declare compliance above 3.16.0.
1. It will promote usage of `isVisible` (and other deprecations that fit the same criteria)
to a runtime assertion, assuming that these usages are unexpected.

If the addon were to change its devDependencies to `~3.17.0`, _more_ deprecations
would fall into this category, without the addon author needing to update anything
else. In this way, the `"auto"` configuration allows addon authors to give parent
apps the best scenario, without having to micromanage another config.

Lastly, the addon can explicitly state that it does _not_ comply with the `isVisible`
deprecation by setting their compliance to a SemVer version (similar to how an
application would declare the same):

```json
{
  "devDependencies": {
    "ember-source": "~3.16.0"
  },
  "ember": {
    "deprecations": {
      "addons": {
        "ember-source": {
          "stage": "enabled",
          "version": "3.14.3"
        }
      }
    }
  }
}
```

In this config, the `ember-source` version installed in `devDependencies` is
irrelevant and is ignored.

This config tells Ember that parent apps _cannot_ declare themselves to be in
compliance with even _available_ deprecations in 3.15.0, because they install
a dependency that is not compliant. Ember will throw a build time error for
parent apps that attempt to do this. For example, this config will throw an error:


```json
{
  "ember-source": {
    "stage": "available",
    "version": "3.15.0"
  }
}
```

This means that apps that use dependencies that have pinned their compliance to
an older version, must either contribute to the dependency or fork it.

There are some rare cases in which a addon declares its compliance to a version
due to a deprecation it cannot address, but that part of the addon is not used
by the parent app. This RFC does _not_ attempt to solve this use case as deprecation
compliance is namespaced by the name of the package, rather than individual features.

### New apps and addons

App blueprints will be updated to include the following config:

```json
{
  "ember": {
    "deprecations": {
      "addons": {
        "ember-source": {
          "stage": "enabled",
          "version": "<%= version %>"
        }
      }
    }
  }
}
```

Addon blueprints will be updated opt addons into all `available` deprecations
with this config:

```json
{
  "ember": {
    "deprecations": {
      "addons": {
        "ember-source": {
          "stage": "available",
          "version": "auto"
        }
      }
    }
  }
}
```

### Existing Method of handling deprecations

Today, apps have three mechanisms for handling deprecations:

1. `registerDeprecationHandler()`: which allows apps a custom way to handle deprecations
1. `Ember.ENV.RAISE_ON_DEPRECATION` configuration, which promotes deprecations to assertions
1. `ember-cli-deprecation-workflow` which makes it easier to work on one deprecation at a time

This RFC treats these as out of scope, but also should not affect how they work.
Future RFCs may address them individually.

## How we teach this

Advancing a deprecation through stages is conceptually easy to understand.
Application and addon developers who have no knowledge of this RFC should be
able to read a deprecation message and understand both that "available" deprecations
are not urgent and "enabled" deprecations require an action. This should be
reflected in the deprecation message that is logged.

New applications will have a much easier time understanding this as blueprints
will be updated to include some default configurations.

There will need to be new Guides for existing apps to be able to understand
new deprecations more easily. The deprecation message should log links to these
guides.

Some places in the guides that will have to be updated:

- https://cli.emberjs.com/release/writing-addons/deprecations/
- https://guides.emberjs.com/release/configuring-ember/handling-deprecations/
- https://guides.emberjs.com/release/configuring-ember/debugging/#toc_dealing-with-deprecations

## Drawbacks

- Confusion about what the config means.

## Alternatives

Not sure.

## Unresolved questions

- How does addon config interact with the dummy app, since there is only one package.json?

[1]: https://github.com/emberjs/ember.js/blob/master/CHANGELOG.md#v3150-december-9-2019
