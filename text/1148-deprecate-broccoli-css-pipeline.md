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

With vite becoming default we should use it's CSS pipeline as the default experience. We can advise in a deprecation guide that `/@embroider/virtual/app.css` exists if they still need to use it but new apps should be using the vite CSS pipeline by default. This would only affect Vite apps, classic build apps will still use the broccoli CSS pipeline, we would not be removing the functionality from ember-cli, just letting vite process application CSS files.

## Motivation

No longer need to maintain ember specific CSS tooling like ember-cli-sass ember-css-modules etc. Vite comes with sane defaults to use postcss, sass, less, stylus etc out of the box.
We also wont have to keep explaining to people how to opt in to vite's CSS pipeline.

## Transition Path

This can be implemented initially by simply replacing the CSS link tags in our app blueprint to load the file on disk. This would be done in the app index.html and the test index.html.

```html
<!-- replace this -->
<link integrity="" rel="stylesheet" href="/@embroider/virtual/app.css">

<!-- with this -->
<link integrity="" rel="stylesheet" href="/app/styles/app.css">
```

We can output a deprecation warning in the CLI if the virtual file is used.

We would want to eventually remove the virtual file from our vite compat plugins.

Deprecation guide wording might be something like this:

If you are using addons like `ember-cli-sass` and `ember-css-modules` you can update your `index.html` files to point at `/@embroider/virtual/app.css` instead of `/app/styles/app.css` to give you time to migrate to the vite pipeline. 

```html
<!-- replace this -->
<link integrity="" rel="stylesheet" href="/app/styles/app.css">

<!-- with this -->
<link integrity="" rel="stylesheet" href="/@embroider/virtual/app.css">
```

We would encourage you to migrate to the vite pipeline by adopting the right tooling as covered by https://vite.dev/guide/features.html#css-pre-processors

You can see examples of different CSS tooling being used in this repository https://github.com/evoactivity/ember-vite-css-examples/

## How we teach this

The guide docs only assume working with plain CSS, this content would be unaffected by this change.
There is nothing in the API docs that would need updating.
The change should be transparent for people using plain CSS.

## Drawbacks

None that I can think of.

## Alternatives

Add a guide to the docs on how to opt in to vite's CSS pipeline.





