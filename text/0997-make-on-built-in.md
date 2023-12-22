---
stage: accepted
start-date: 2023-12-22T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - framework
  - learning
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/997
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

# Make `{{on}}` a built in modifier  

## Summary

Today, when using gjs/gts/`<template>`, in order to bind event listeners, folks _must import_ the `{{on}}` modifier.
Because event listening is so commonplace, this is a grating annoyance for developers.

This RFC proposes that `{{on}}` be built in to `glimmer-vm` and not require importing.

## Motivation

Given how common it is to use the `{{on}}` modifier:

```gjs
import { on } from '@ember/modifier';

<template>
    <button {{on 'click' @doSomething}}>
        click me
    </button>

    <form {{on 'submit' @localSubmit}}>
        <label
            {{on 'keydown' @a}}
            {{on 'keyup' @a}}
            {{on 'focus' @a}}
            {{on 'blur' @a}}
        >
        </label>

        <button>
            submit
        </button>
    </form>
</template>
```

It should be built in to the templating engine, Glimmer, so that folks don't need to import it.

There is precedence for this already as the following are already commonplace and built in:
- `(helper)`
- `(modifier)`
- `(component)`

It's historically been the stance that, 

"If it can be built in userspace, it should be, leaving the framework-y things to be only what can't exist in userspace"

But to achieve the ergonomics that our users want, we should provide a more cohesive experience, rather than require folks import from all of (in the same file):
- `@glimmer/component`
- `@glimmer/tracking`
- `@ember/modifier`
- `@ember/helper`
- `ember-modifier`
- `ember-resources`
- `tracked-built-ins`

Some of the above may unify in a separate RFC, but for the template-utilities, since the modules that provide them are already so small, it makes sense to inherently provide them by default. Especially as we can target strict-mode only, so that we don't run in to the same implementation struggles that built-in [Logical Operators](https://github.com/emberjs/rfcs/pull/562), [Numeric Comparison Operators](https://github.com/emberjs/rfcs/pull/561), and [Equality Operators](https://github.com/emberjs/rfcs/pull/560) have faced.

<details><summary>some context on those RFCs</summary>

The main problem with adding default utilities without strict-mode is that it becomes very hard to implement a way for an app to incrementally, and possibly per-addon, or per-file, to adopt the default thing due to how resolution works. Every usage of the built in utility would also require a global resolution lookup (the default behavior in loose mode templates) to see if an addon is overriding the built ins -- and then, how do you opt in to the built ins, and _not_ let addons override what you want to use?

With gjs/gts/`<template>`, this is much simpler, as in strict-mode, you can check if the scope object defines the helpers, and if not, use the built in ones.

This strategy of always allowing local scope to override default-provided utilities will be a recurring theme.

</details>


## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

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
