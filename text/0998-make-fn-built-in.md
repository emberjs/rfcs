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
  accepted: https://github.com/emberjs/rfcs/pull/998
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

# Make `(fn)` a built in helper 

## Summary

Today, when using gjs/gts/`<template>`, in order to bind event listeners, folks _must import_ the `(fn)` modifier.
Because partial application is so commonplace, this is a grating annoyance for developers.

This RFC proposes that `(fn)` be built in to `glimmer-vm` and not require importing.

## Motivation

There is precedence for `fn` being built in, as all the other partialy-application utilities are already built in.

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

---------------

_Making `fn` a built-in will help make writing components feel more cohesive and well supported, as folks will not need to cobble together many imported values_

----------------


## Detailed design

This change would affect strict-mode only. This is so that today's existing code that imports `fn` from `@ember/helper` will still work due to how values define locally in scope override globals.

The behavior of `fn` would be the same as it is today, but defined by default in the `glimmer-vm`.

## How we teach this

Once implemented, the guides, if they say anything about gjs/gts/`<template>` and `fn` by the time this would be implemented, would only remove the import.

## Drawbacks

People may not know where `fn` is defined.
- counterpoint: do they need to?

## Alternatives

n/a

## Unresolved questions

n/a
