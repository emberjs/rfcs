---
stage: accepted
start-date: 2025-10-11T16:34:13.823Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1148
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

<!-- Replace "RFC title" with the title of your RFC -->

# Deprecate broccoli CSS pipeline

## Summary

With vite becoming default we should use it's CSS pipeline as the default experience. We can advise in a deprecation guide that `/@embroider/virtual/app.css` exists if they still need to use it but new apps should be using the vite CSS pipeline by default.

## Motivation

No longer need to maintain ember specific CSS tooling like ember-cli-sass ember-css-modules etc. Vite comes with sane defaults to use postcss, sass, less, stylus etc out of the box.
We also wont have to keep explaining to people how to opt in to vite's CSs pipeline.

## Detailed design

Replace the CSS import in our app blueprint to the file on disk.

```html
<!-- replace this -->
<link integrity="" rel="stylesheet" href="/@embroider/virtual/app.css">

<!-- with this -->
<link integrity="" rel="stylesheet" href="/app/styles/app.css">
```

## How we teach this

As part of a deprecation guide explain how to keep using the broccoli pipeline whilst they migrate (using `/@embroider/virtual/app.css`)

## Drawbacks

None that I can think of.

## Alternatives

Add a guide to the docs on how to opt in to vite's CSS pipeline.

