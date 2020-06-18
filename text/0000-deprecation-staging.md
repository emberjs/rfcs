- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Deprecation Staging

## Summary

Establish a staging system for deprecations that allows Ember and other
libraries to roll them out more incrementally. This system will also enable
users to opt-in to a more strict usage mode, where deprecations will assert if
encountered.

The initial stages would be:

* Available
* Required

## Motivation

Ember has a very robust deprecation system which has served the community very
well. However, there are a number of pain points and use cases that it does not
cover very well:

* **Ecosystem Adoption**: Generally, deprecations need to be adopted by the
  addon ecosystem before they can be adopted by applications. This is because
  apps may see a deprecation that is triggered by addon code, that they have no
  control over. However, today there is no way to add a deprecation just for
  addons - adding a deprecation will make it appear everywhere at once.

  This creates pressure to only add deprecations when most of the community has
  already transitioned away from the API. However, it also means that we can't
  use a strong signal, like a deprecation, to let the community know that it
  _should_ begin transitioning. This cyclic dependency can make it difficult to
  push the community forward over time.

  Deprecation staging would allow deprecations to be added incrementally.
  Combined with default settings for apps and addons that enable deprecations at
  different stage levels, this would allow incremental rollout throughout the
  ecosystem.

* **Pacing**: Currently, deprecations have to have their timeline built
  into them from the get go. The deprecation API requires an `until` version,
  and there isn't much flexibility during the rollout process because of this.
  Once a deprecation is RFC'd and merged, there isn't much the community can do
  to slow down the adoption of the process.

  Staging would give the community multiple serialization points to decide
  whether or not a deprecation was ready to move forward in the process, and to
  update the timeline if so. This would allow a deprecation to slow down or
  speed up based on real world feedback and adoption.

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
  again in the future. Since deprecations only produce a console warning and not
  an error, there isn't a way to prevent this from happening today.

  With staging, users will be able to opt-in to deprecations becoming
  assertions at a particular stage, in a non breaking way. This will allow users
  to prevent backsliding as they incrementally migrate their applications to new
  APIs. It will also provide the basis for shaking those deprecated features -
  if they throw an error when used, they can't be used, so they can also be
  removed and the app will continue to work.

## Dependencies

This RFC depends on [RFC Staging](https://github.com/emberjs/rfcs/pull/617). The
assumption is that the deprecation will advance in stages as their RFCs advance.
The relationship won't necessarily be 1-to-1, but the important thing
process-wise is that community members will have the ability weigh in via FCP
each time the deprecation is ready to advance to the next stage.

## Detailed design

There are two stages that are proposed for deprecations in this RFC:

1. **Available**: Deprecations are initially released in the available stage.
   These deprecations are available to Ember users at this point, but given a
   lower priority since they were just released. Available deprecations are
   enabled by default in addons, but not in apps.

   The Available stage being released should coincide with the RFC for the
   deprecation moving into "Released" stage.

2. **Required**: Required is the final stage for deprecations. Deprecations
   should be moved into Required after the replacement APIs have moved into
   Recommended in the RFC process, and there is generally high confidence that
   the addon ecosystem has removed the deprecated API for the most part.
   Required deprecations cannot be disabled.

   The Required stage being released should coincide with the RFC for the
   deprecation moving into the "Recommended" stage.

Deprecations will progress through these stages linearly. In the future, more
stages could be added, but the progression will remain linear.


There are places where APIs and configs will need to be updated to add stages:

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

```ts
interface DeprecationOptions {
  id: string;
  for: string;
  until: string;
  availableSince: string;
  requiredSince?: string;
  url?: string;
}
```

Stepping through the new options individually:

* `for`: This is used to indicate the library that the deprecation is for.
  `deprecate` is a public API that can be used by anyone. Now that we intend to
  show or hide deprecations based on their stage, we need to have some more
  information about what library the deprecation belongs to. It should be the
  package name for the library.

* `availableSince`: This property should contain an exact SemVer version. This is
  the first version that the deprecation existed in while being in the Available
  stage. If this property is present, and `requiredSince` is not, it indicates
  that the deprecation is in the **Available** stage.

  This property should continue to exist even after `requiredSince` is added,
  and the deprecation advances to the next stage. If it is not passed, it will
  throw an error. This property is used for deprecation compliance assertions,
  which will be discussed more below.

* `requiredSince`: This property should contain an exact SemVer version. This is
  the first version that the deprecation existed in while being in the Available
  stage. If this property exists, then it indicates that the deprecation is in
  the **Required** stage.

### Configuration

Users can configure their application or addon using the `ember.deprecations`
property in `package.json`. This property will be empty by default, but can be
customized like so:

```json
{
  "ember": {
    "deprecations": {
      "ember-source": {
        "stage": "available",
        "complianceStage": "required",
        "complianceVersion": "3.16.0"
      },
      "@ember/data": "required"
    }
  }
}
```

The property itself is an object, with a keys that are package names (the `for`
that was provided to `deprecate` above). Values can be:

1. The name of the stage to begin show deprecations at for that library. For
   instance, `"available"` will should all Available _and_ Required deprecations
   for the library.

   Valid stage names are `"available"` and `"required"`. The default value if
   one is not provided is `"required"`.

2. An object containing the following keys:

   * `"stage"`: The stage that deprecations should begin showing at, same as
     above.
   * `"complianceStage"`: The stage that deprecations are compliant with. This
     is discussed in more detail below, in the Deprecation Compliance section.
   * `"complianceVersion"`: The version that the deprecations are compliant
     with. This is discussed in more detail below, in the Deprecation Compliance
     section.

### Deprecation Compliance

Users can opt-in to some deprecations becoming assertions. They do this by
stating that their application is _compliant_ with a given library's
deprecations as of a certain stage and version.

What that means, is that they guarantee that they are not using _any_ deprecated
APIs that were in _the given stage_ prior to or including _the given version_.
If they _do_ use an API that was deprecated prior to then, `deprecate` will
throw an error instead of logging a warning. The error will contain the same
message, with an additional note that the user is encountering this error
because of their deprecation compliance settings.

Deprecation compliance cannot be specified for a _future_ version of the
library. This is to prevent breaking changes as new deprecations are added to
the library. We won't know what the exact deprecations are until a version is
released, so we can't specify compliance with it. Attempting to do this will
throw an error.

### Usage and Config Example

Let's say that we use `deprecate` to add a new deprecation to our library. It's
a brand new deprecation, so we're going to make it Available.

```js
deprecate('this feature is deprecated!', false, {
  id: 'my-awesome-library.my-awesome-feature',
  for: 'my-awesome-library',
  availableSince: '1.2.3',
  until: '2.0.0'
});
```

These configurations would not log a warning or throw an error:

```json
// No config value provided, defaults to "required" which is
// a later stage than the deprecation is in currently
{
  "ember": {}
}
```
```json
// Config value provided, set to "required" which is
// a later stage than the deprecation is in currently
{
  "ember": {
    "deprecations": {
      "my-awesome-library": "required"
    }
  }
}
```
```json
// Config value provided, set to "required" which is
// a later stage than the deprecation is in currently
{
  "ember": {
    "deprecations": {
      "my-awesome-library": {
        "stage": "required"
      }
    }
  }
}
```
```json
// Compliance config is set to a later stage than the
// deprecation is currently in, even though the version
// is set to the current version or lower.
{
  "ember": {
    "deprecations": {
      "my-awesome-library": {
        "complianceStage": "required",
        "complianceVersion": "1.2.3"
      }
    }
  }
}
```
```json
// Compliance config is set to the current stage, but
// the version is set to a lower version than the one that
// the deprecation was introduced in.
{
  "ember": {
    "deprecations": {
      "my-awesome-library": {
        "complianceStage": "available",
        "complianceVersion": "1.0.0"
      }
    }
  }
}
```

These configurations would log a warning:

```json
// Config provided, and its set to the current stage
// that the deprecation is in.
{
  "ember": {
    "deprecations": {
      "my-awesome-library": "available"
    }
  }
}
```
```json
// Config provided, and its set to the current stage
// that the deprecation is in.
{
  "ember": {
    "deprecations": {
      "my-awesome-library": {
        "stage": "available"
      }
    }
  }
}
```

These configurations would throw an error:

```json
// Compliance config is set to the current stage, and
// the version is set to a version that is greater than or
// equal to the version that the deprecation was added in.
{
  "ember": {
    "deprecations": {
      "my-awesome-library": {
        "complianceStage": "available",
        "complianceVersion": "1.2.3"
      }
    }
  }
}
```


## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
