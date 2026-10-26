---
stage: released
start-date: 2025-05-01T00:00:00.000Z
release-date: 2025-10-13
release-versions:
  ember-source: 6.8.0
teams:
  - framework
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1099'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/1128'
  released: 'https://github.com/emberjs/rfcs/pull/1144'
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
         * Optionally configure the rendering environment
         */
        env?: { 

            /**
             * When false, modifiers will not run.
             */
            isInteractive?: boolean; 

            /**
             * All other options are forwarded to the underlying renderer.
             * (its API is currently private and out of scope for this RFC, 
             *  so passing additional things here is also considered private API)
             */
            [rendererOption: string]?: unknown;
        };

        /**
         * These args get passed to the rendered component
         *
         * If your args are reactive, re-rendering will happen automatically.
         * 
         */
        args?: Record<string, unknown>;
  }
): RenderResult | undefined {
    /* ... implementation details ... */
}
```

`RenderResult` is a subset of the currently pre-existing `interface` from `@glimmer/interfaces`, but would be exposed as public API via the only return type of `renderComponent`.

Its shape is:
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

#### `env.isInteractive` (defaults to true)

When isInteractive is false, modifiers don't run, when true, modifiers do run.

#### `into`

The element to render the component into.

#### `args`

The args to pass to the component

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
import { getOwner } from '@ember/owner';
import { renderComponent } from '@ember/renderer';

const Demo = <template>hi</template>;
const render = modifier((element) => {
    let owner = getOwner(); // hypothetical getOwner that works in function modifiers
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

`renderComponent` _is_ a low-level API, but its use cases are powerful:

### Micro Applications

In _any_ framework, or _non_-framework, we incidentally re-enable the ability to use [JSBin](https://jsbin.com) or similar tools.

Combined with [RFC #931](https://github.com/emberjs/rfcs/pull/931), we have a completely buildless framework.

For example:

```html
<script type="importmap">
{
  "imports": {
    "@ember/renderer": "https://esm.sh/*ember-source/dist/packages/@ember/renderer/index.js",
    "@ember/template-compiler": "https://esm.sh/*ember-source/dist/packages@ember/template-compiler",
    // etc
  }
}
</script>

<script type="module">
  import { template } from '@ember/template-compiler';
  import { renderComponent } from '@ember/renderer';

  const Demo = template('hi');
  renderComponent(Demo); // default "into" is the document.body
</script>

<body></body> <!-- "hi" is rendered here -->
```

### REPL

Combined with [RFC #931](https://github.com/emberjs/rfcs/pull/931), we can dynamically compile and render components from user input.

```html
<script type="module">
  import { template } from '@ember/template-compiler';
  import { renderComponent } from '@ember/renderer';

  let component;
  let cleanup;

  document.querySelector('textarea').addEventListener('change', (event) => {
      if (cleanup) cleanup();

      component = template(event.target.value);
   
      if (component) { 
        let result = renderComponent(component, { into: document.querySelector('#app') });
        cleanup = () => result.destroy();
      }
  });

</script>
<body>
  <textarea>template contents</textarea>
  <div id="render-output"></div>
</body>
```

> [!NOTE]
> Depending on your application, you may want to consider sanitizing user input, so users don't inject their own script / style tags into your app.
> See: [DOMPurify](https://github.com/cure53/DOMPurify) and [Web Sanitizer API](https://developer.mozilla.org/en-US/docs/Web/API/Sanitizer) and [TrustedHTML](https://developer.mozilla.org/en-US/docs/Web/API/TrustedHTML)

### Integration with "islands"-based documentation tools

These include Vitepress, Astro, etc. 

> [!NOTE]
> We can already make integrations with [vitepress](https://vitepress.dev/), [astro](https://astro.build/), etc, using code similar to [here in repl-sdk](https://github.com/NullVoxPopuli/limber/blob/06a65df246f147bf085ae5240a94d81455616e22/packages/repl-sdk/src/compilers/ember/render-app-island.js#L76). But `renderComponent` is a more streamline, composable, and user-friendly approach to rendering subtrees of ember code.

<details><summary>hypothetical Astro integration implementation</summary>

This hypothetical implementation _isn't tested_, and is based on [`@astrojs/svelte`](https://github.com/withastro/astro/blob/3276b798d4ecb41c98f97e94d4ddeaa91aa25013/packages/integrations/svelte/src/index.ts#L5) at the time of writing.

> [!IMPORTANT]
> V1 Addons not supported here. This uses "minimal app" concepts, which we haven't talked about much publicly, but they are present in the [v2 addon blueprint overe here](https://github.com/ember-cli/ember-addon-blueprint/) (for testing, "docs app", etc).

```ts
// astro/packages/integrations/ember/src/index.ts

import { extensions, ember } from '@embroider/vite';
import { babel } from '@rollup/plugin-babel';
import type { AstroIntegration, AstroRenderer, ContainerRenderer } from 'astro';

function getRenderer(): AstroRenderer {
	return {
		name: '@astrojs/ember',
		clientEntrypoint: '@astrojs/ember/client.js',
		serverEntrypoint: '@astrojs/ember/server.js',
	};
}

export function getContainerRenderer(): ContainerRenderer {
	return {
		name: '@astrojs/ember',
		serverEntrypoint: '@astrojs/ember/server.js',
	};
}

export default function emberIntegration(options?: Options): AstroIntegration {
	return {
		name: '@astrojs/ember',
		hooks: {
			'astro:config:setup': async ({ updateConfig, addRenderer }) => {
				addRenderer(getRenderer());
				updateConfig({
					vite: {
						optimizeDeps: {
							include: ['@astrojs/ember/client.js'],
							exclude: ['@astrojs/ember/server.js'],
						},
						plugins: [
                            ember(),
                            babel({ babelHelpers: 'inline', extensions }), // NOTE: it may be worth inlining a default config here
                        ],
					},
				});
			},
		},
	};
}

export { vitePreprocess };
```

```ts
// astro/packages/integrations/ember/src/client.ts

export default function (element: HTMLElement) => {
  return async (Component: any, props: Record<string, any>, ...rest: unknown[]) => {
    let result = renderComponent(Component, { into: element, args: props, /* ... */  });
    element.addEventListener('astro:unmount', () => result.destroy(), { once: true });
  }
}
```

</details>

## Potential Nuances

All renderComponents rendered within an ember app would share their reactivity. Since the passed `args`'s reactivity can cause the contents of `renderComponent` to re-render.

Outside of ember projects, _currently_ if multiple `renderComponent`s are used from different ember-source versions (possible if someone pulls in, for example, two web components from different libraries which internally use `renderComponent`), the `renderComponent` usages would not be co-reactive with each other's data. This is something that would need to be resolved in the underlying _renderer_ and is out of scope for discussion in this RFC.

Repeat calls to `renderComponent` with the same element will _append_ all contents on that element. This means if someone wants multiple apps to replace one another, they should manually manage the destruction of the previously rendered app before calling `renderComponent` with the same element again. This is to be consistent with `document.body` vs within an app's component's DOM content. 

If apps are replaced by other apps (such as rendering into the same element multiple times), there is a risk of memory leak if the prior apps were not destroyed, and _if_ those previous apps have global state that would _need_ cleanup (global event listeners, etc).


## Drawbacks

n/a

## Alternatives

There is an existing implementation in [PR#20781](https://github.com/emberjs/ember.js/pull/20781/)

Here is where this RFC differs from that implementation

- proposed return type is much narrower than what is exposed (not the whole `RenderResult` from `@glimmer/interfaces`)
    - returned object from `renderComponent` also has `destroy` on it, for convenience
- Configuration all has **default values**, so that basic runtime usage is as simple as possible:
    - isInteractive is optional and defaults to `true`
    - owner is optional and defaults to a private empty object (`{}`)
    - env is optional (as all its contents are optional)
- removed features (can be added later if we need)
    - removed `document` from the env, as we already pass in `into`, and `into` cannot be from a different document.
        ```js
        document.contains($0) // $0 is an element selected in the browser's element debugger
        // >  true
        document.contains(document.createElement('div'))
        // > false
        ```
        so if `false === document.contains(into)`, then we can get the correct `document` via `into.ownerDocument` 
    - removed `hasDOM` -- although for the current implementation defaults to true -- (we mostly have a DOM, but this could be a useful utility (or a derived value from the overall environment -- as would probably shake out of work on [Swappable Renderers](https://github.com/emberjs/ember.js/issues/20648) -- like, defining what `createElement` means in a terminal environment, for example). For the purposes of this RFC, it is "private API" that could be passed to the underlying renderer.
    - removed `parentElement` from the returned object f rom `renderComponent`
    - removed `alwaysRevalidate` (and the whole options object) from `rerender`) -- as I couldn't find evidence of it being used in the implementation PR -- can always be added later if we need it.
    - removed `rerender` -- we want to encourage reactivity

## Unresolved questions

- what happens if `args` is a `TrackedMap` -- what happens if new keys are added or deleted?
    - it is not intended that `renderComponent` becomes a back door for splat-arguments. We expect the keys within args are static, as they are in other ember code. (There is on-going investigation in to formally supporting splat-arguments everywhere, and when that happens, perhaps a `TrackedMap` can be supported then)
