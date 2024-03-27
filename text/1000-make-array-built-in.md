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
  accepted: https://github.com/emberjs/rfcs/pull/1000
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

# Make `(array)` a built in helper 

## Summary

Today, when using gjs/gts/`<template>`, in order make arrays in templates, folks _must import_ the `(array)` helper.
Because creating arrays is fairly commonplace, this is an annoyance for developers, in part, due to how almost every other language has array literal syntax.

This RFC proposes that `(array)` be built in to `glimmer-vm` and not require importing.

## Motivation

Arrays and Objects are not only very common to create, they are essential tools when yielding data out of components.

There is alternate motivation to implement _literals_ for arrays and objects, but that is a bigger can of worms for another time.

## Detailed design

This change would affect strict-mode only. This is so that today's existing code that imports `array` from `@ember/helper` will still work due to how values defined locally in scope override globals.

The behavior of `array` would be the same as it is today, but defined by default in the `glimmer-vm`.

Being built in can give folks confidence that each element in the array is individually reactive.

## How we teach this

Once implemented, the guides, if they say anything about gjs/gts/`<template>` and `array` by the time this would be implemented, would only remove the import.

The guides should also detail which functions are built in to the framework and, therefore, do not need to be imported.

## Drawbacks

People may not know where `array` is defined.
- counterpoint: do they need to?

## Alternatives

n/a

## Unresolved questions

n/a
