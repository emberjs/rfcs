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

Building an app from scratch requires a fair amount of boilerplate today, and `renderComponent` abstracts a minimal API that would allow easy integration of components in other enviroments, such as [Astro](https://astro.build/), or [Vitepress](https://vitepress.dev/), or [Storybook](https://storybook.js.org/), or runtime demos such as [JSBin](https://jsbin.com/).

This also pairs well with [RFC#931: JS Representation of Template Tag](https://github.com/emberjs/rfcs/pull/931), in that both features together give folks enough to use Ember without compilation (compilation still recommended for optimizations, however).


## Detailed design

Users would import `renderComponent` from `@ember/renderer` (a pre-existing module)

The interface:
```ts
/**
 * Renders a component into an element, given a valid component definition.
 */
export function renderComponent(
    /**
     * The component definition to render.
     *
     * Any component that has had its manager registered is valid.
     * For the component-types that ship with ember, manager registration 
     * does not need to be worried about. 
     */
    component: object,
    options: {
        /**
         * The element to render the component in to.
         */
        into: Element;

        /**
         * Optional owner. Defaults to `{}`, can be any object, but will need to implement the [Owner](https://api.emberjs.com/ember/release/classes/Owner) API for components within this render tree to access services.
         */
        owner?: object;

        /**
         * Configure the `document` and `isInteractive`
         */
        env?: { 
            /**
             * Defaults to globalThis.document.
             */
            document?: SimpleDocument | Document; 

            /**
             * When false, modifiers will not run.
             */
            isInteractive?: boolean; 
        };

        /**
         * These args get passed to the rendered component
         *
         * If your args are reactive, re-rendering will happen automatically.
         */
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
     * Destroys the render tree and removes all rendered content from the element rendered into.
     */
    destroy(): void
}
```

When called, updates will be scheduled and fully interactive components will work as folks would intuitively expect.
The details of _how_ are out of scope for this RFC, and could change as we make improvements to rendering.

### The Parameters

#### `owner` (defaults to `{}`)

some object to represent the `Owner` (if not the host-app's `Owner`).

#### `env.document` (default's to globalThis.document)

The browser's `document`, for accessing needed APIs for rendering and managing DOM

#### `env.isInteractive` (defaults to true)

When isInteractive is false, modifiers don't run, when true, modifiers do run.

#### `into`

The element to render the compnoent into.

#### `args`

The args to pass to the component

### Behaviors

TODO: 
- rendering multiple renderComponent on the page
  - who creates tracking frames / how do they interop? all of them? are they coordinated?
- shared reactive state between multiple renderComponents
- can the ember-in-ember use case extend the owner in subtrees?

### Usage

via `template()` in JSBin:

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

rendering ember in React:
```jsx
import { renderComponent } from '@ember/renderer';
import { template } from "@ember/template-compiler";
import React, { useEffect, useRef } from 'react';

const GreetingComponent = template(`Hello World`);

const RunOnceComponent = () => {
    const myRef = useRef(null);

    useEffect(() => {
        if (myRef.current) {
            let result = renderComponent(GreetingComponent, {
                into: myRef.current,
            });

            return () => result.destroy(); 
        } 
    }, []); 

    return (
        <div ref={myRef}></div>
    );
};

export default RunOnceComponent;
```

if we wanted to render ember in ember via modifier (silly):
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

    return () => result.destroy();
});


<template>
    <div {{render}}></div>
</template>
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
- returned object from `renderComponent` also has `destroy` on it, for convenience
- removed `parentElement` from the returned object f rom `renderComponent`
- removed `alwaysRevalidate` (and the whole options object) from `rerender`) -- as I couldn't find evidence of it being used in the implementation PR -- can always be added later if we need it.
- removed `rerender` -- we want to encourage reactivity

## Unresolved questions

n/a
