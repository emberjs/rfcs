---
stage: accepted
start-date: 2025-05-01T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1099 
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

# `renderComponent` 

## Summary

Add a new API to for rendering components into any DOM element.

## Motivation

Building an app from scratch requires a fair amount of boilerplate today, and `renderComponent` abstracts a minimal API that would allow easy integration of components in other enviroments, such as [Astro](https://astro.build/), or [Vitepress](https://vitepress.dev/), or runtime demos such as [JSBin](https://jsbin.com/).

This also pairs well with [RFC#931: JS Representation of Template Tag](https://github.com/emberjs/rfcs/pull/931), in that both features together give folks enough to use Ember without compilation (compilation still recommended for optimizations, however).


## Detailed design

Users would import `renderComponent` from `@ember/renderer` (a pre-existing module)

The interface:
```ts
export function renderComponent(
  component: object,
  {
    owner,
    env,
    into,
    args,
  }: {
    owner?: object;
    env?: { document?: SimpleDocument | Document; isInteractive?: boolean; };
    into: IntoTarget;
    args?: Record<string, unknown>;
  }
): RenderResult | undefined {
    /* ... implementation details ... */
}
```

`RenderResult` is a subset of the currently a pre-existing `interface` from `@glimmer/interfaces`, but would be exposed as public API via the only the return type of `renderComponent`.

It's shape is:
```ts
export interface RenderResult {
    /**
    * The element rendered in to 
    */
  parentElement(): SimpleElement;
    /**
    * Re-renders the component
    */
  rerender(options?: { alwaysRevalidate: false }): void;
}
```

### The Parameters


#### `owner`

some object to represent the `Owner` (if not the host-app's `Owner`).

#### `env.document` (default's to globalThis.document)

The browser's `document`, for accessing needed APIs for rendering and managing DOM

#### `env.isInteractive` (defaults to true)

When isInteractive is false, modifiers don't run, when true, modifiers do run.

#### `into`

The element to render the compnoent into.

#### `args`

The args to pass to the component

### Usage

via modifier:
```gjs
import { modifier } from 'ember-modifier';
import { destroy } from '@ember/destroyable';
import { renderComponent from '@ember/renderer';

const Demo = <template>hi</template>;
const render = modifier((element) => {
    let result = renderComponent(Demo, {
        owner,
        env: { document: document, isInteractive: true },
        into: element,
    });

    return () => destroy(result);
});


<template>
    <div {{render}}></div>
</template>
```

via `template()`:

```html
<DOCTYPE html>
<html>
    <head>
        <title>Demos can be simple again</title>
    </head>
<body>
    <div id="my-element"></div

    <script>
        import { renderComponent } from 'https://esm.sh/@ember/renderer';
        import { template } from "https://esm.sh/@ember/template-compiler";

        renderComponent(template(`Hello World`), {
            into: document.querySelector('#my-element'),
        });
    </script>
</body>
</html>
```

## How we teach this

`renderComponent` is meant as an integration-enabling API and would not be added to the guides, or need any documentation beyond what would live in source on the function itself.

## Drawbacks

n/a

## Alternatives

There is an existing implementation in [PR#20781](https://github.com/emberjs/ember.js/pull/20781/)

Here is where this RFC differs:
- no hasDOM (we always have a DOM, as we have not done any design on [Swappable Renderers](https://github.com/emberjs/ember.js/issues/20648))
  - different `renderComponent` implementations could be what we swap out for whole-sale different renderers (rather than feature-zebra-striping them) 
- proposed return type is much narrower than what is exposed (not the whole `RenderResult` from `@glimmer/interfaces`)
- isInteractive is optional and defaults to `true`
- document is optional and defaults to `globalThis.document`
- env is optional (as all its contents are optional)
- owner is optional and defaults to a private empty object (`{}`)

## Unresolved questions

n/a
