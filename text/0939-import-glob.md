---
stage: ready-for-release
start-date: 2023-07-27T00:42:02.085Z
release-date:
release-versions:
teams:
  - cli
  - framework
  - learning
  - typescript
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/939'
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

# Introduce a Wildcard Module Import API

## Summary

Introduce `import.meta.glob()` for use in all Ember apps and addons.

## Motivation

This RFC is siblings with [an RFC](https://github.com/emberjs/rfcs/pull/938) that deprecates all usage of Ember's traditional AMD infrastructure. That necessarily means we will remove `requirejs.entries` and `requirejs._eak_seen`. So we need to explain what you're supposed to use instead if you need to enumerate modules. `import.meta.glob()` is one answer to that question.

## Detailed design

First, an illustrative example:

```js
// If you type this in your app:
const widgets = import.meta.glob('./widgets/*.js')

// It gets automatically converted into something like this:
const widgets = {
  './widgets/first.js': () => import('./widgets/first.js'),
  './widgets/second.js': () => import('./widgets/second.js'),
}
```

This design builds off [Vite's Glob Import Feature](https://vitejs.dev/guide/features.html#glob-import), but since that feature is non-standard and offers a rather wide API surface area, we're picking a well-defined subset that we're committed to supporting, not just under Vite but under all build tooling that we support, including today's classic builds.

`import.meta.glob()` is a special function that:
 - can only be used with a string-literal first argument
 - can only be invoked directly as `import.meta.glob()`. You cannot pass it as a value, nor can you pass `import.meta` as a value and then try to call `.glob()` on that.

The string argument **must** start with `./` or `../`. Only relative imports are supported.

Escaping your own package via repeated `../` is not allowed.

 - For Ember apps, your package is the `/app` directory. 
 - For classic (v1) Ember addons, your package is the `/addon` directory.
 - For v2 Ember Addons, your package is your actual NPM package.

### Pattern Matching Specifics

The pattern always uses `/` as the path separator, regardless of operating system.

No automatic file extension resolution is performed -- to match `.js`, your pattern should end in `.js`.

 - You should think of this happening *before* any transpilation renames your files. If you're authoring typescript, you should write `import.meta.glob('./files/*.ts')` and it will give you back an object whose keys also end in `".ts"`. Similarly, when targeting `.gjs` or `.gts` files you should say so explicitly, like `import.meta.glob('./files/*.gjs')`.

 - However, due to the way template-only components work, you should think of this happening *after* the automatically-created Javascript representation of a template-only component is created. That is, if you want to import a directory full of components, even if some of them are template-only components represented by `.hbs` files, you should still `import.meta.glob('./components/*.js')` as that will match the automatically-created components.

    > An import with an explicit `.hbs` extension has a specific historical meaning that is *not* a component, it's a bare template. You almost never want to do that manually. It's an implementation detail of template co-location and a historical compatibility feature. So if you tried to do `import.meta.glob('./components/*.{js,hbs}')` you would get back a mix of components and things-that-are-not-really-components.
    >
    > Ultimately this is a concern that goes away once you adopt `.gjs` and that would be our recommendation going forward.

`*` matches everything except:
 - path separators 
 - names starting with "." (hidden files)

`**` matches zero or more directories

`?` matches any single character except path separators

`[abc]`: a sequence of characters inside `[]` match any character in that sequence

`{.js,.gjs}`: bash-style brace expansion

### Lazy Mode

By default, `import.meta.glob` gives you asynchronous access to the modules. This is designed to work nicely in systems that lazily load code. However, it does not promise to *introduce* laziness where laziness does not already exist. When building with the classic build pipeline, all your own app code is always included in the bundle regardless of whether anyone imports it or not, and that remains true regardless of whether you use `import.meta.glob()` to access some modules. 

When building with Embroider, you can achieve lazy loading by using `import.meta.glob()` in combination with other features like `staticAppPaths` or `staticComponents`.

The return value from `import.meta.glob()` is the same either way -- you always get functions that return `Promise<Module>`. 

### Eager Mode

`import.meta.glob` supports an optional second argument. The only supported value at this time is the literal `{ eager: true }`.

When eager is true, you get synchronous access to all the modules. These modules cannot be lazily loaded evaluated  -- they will load and evaluate eagerly.

For example:

```js
// If you type this in your app:
const widgets = import.meta.glob('./widgets/*.js', { eager: true })

// It gets automatically converted into something like this:
import _w0 from './widgets/first.js';
import _w1 from './widgets/second.js';
const widgets = {
  './widgets/first.js': _w0,
  './widgets/second.js': _w1,
}
```

### Replacing cross-package usages

Historically, people have used `requirejs.entries` to have complete global access to everywhere from everywhere. `import.meta.glob` is deliberately more restrictive. For example, an addon cannot use `import.meta.glob` to load code out of the consuming application. Instead, addon authors will need to ask apps to pass them what they need.

For example, a future version of `ember-cli-mirage` might tell app authors to put this code into their app and/or tests as a way to dynamically gain access to all the Mirage-specific models, adapters, serializers, etc that the user has written:

```js
import { setup } from 'ember-cli-mirage';

setup(import.meta.glob('./mirage/**/*.js'))
```

Similarly, to do auto-discovery of ember-data models, an existing API like `discoverEmberDataModels()` would now accept them explicitly:

```js
import { discoverEmberDataModels } from 'ember-cli-mirage';

discoverEmberDataModels(import.meta.glob('../models/*.{js,ts}'));
```

### Not allowed in publication format

Addons are free to use `import.meta.glob` in their own code, but our tooling should implement it within the addon's own build, *before* publishing to NPM. `import.meta.glob` is not allowed in published addons on NPM.

This greatly reduces future compatibility concerns, and it doesn't cost us anything in terms of flexibility, given that this spec says `import.meta.glob` is not allowed to cross package boundaries anyway. That is: at publish time, the full list of files that any `import.meta.glob()` expands into is statically known.

### Types

`import.meta.glob` has this signature:

```ts
(pattern: string): Record<string, () => Promise<unknown>>
(pattern: string, { eager: true }): Record<string, unknown>
```

When you know that the things you're importing have a shared interface, it will behoove you to cast to it:

```ts
import { ComponentLike } from '@glimmer/template';

type Button = ComponentLike<{ 
  Args: { "onClick": () => void }, 
  Element: HTMLButtonElement, 
  Blocks: { default: [] } 
}>;

const buttons: Record<string, () => Promise<{ default: Button }>> = import.meta.glob('./buttons/*.js');
```


## How we teach this

We probably do not want to introduce this feature immediately to new users. In typical application development you won't often need it in your own code. So I don't think we need to bring it up in initial tutorial-level content.

In the guides, I think we should add a section titled "ES Modules: Imports and Exports" to the page [Working with HTML, CSS, and JavaScript](https://guides.emberjs.com/release/getting-started/working-with-html-css-and-javascript/). Right now "Modules" exist as a bullet point that directs you to MDN. Similar to what we currently do with classes, we can add a dedicated section with more details.

> ### Modules: Imports and Exports
> Ember apps are authored as JavaScript Modules (also know as "ES Modules"). By convention, your app's modules live in the `/app` directory, so if you see an import like `import Article from "your-app/models/article"`, that is referring to `/app/models/article.js`. If you install dependencies from NPM you can import from them as well.
> 
> In a default Ember app, you can use dynamic `import()` to load third-party modules from NPM on demand, but you can't use it on your own app code. (That feature is available if you're building with [Embroider](https://github.com/embroider-build/embroider/), but that is not the default experience yet.)
>
> If you need to import many modules at once, Ember apps support an extension on top of ES modules called `import.meta.glob()`. For example, `import.meta.glob('./widgets/*.js')` will give you access to all the matching files. You can only use `import.meta.glob()` on files within your own package.

The more detailed nuances of what `import.meta.glob` supports should be taught through good error messages. For example, all of these need to give clear explanation in an error messages:

 - trying to pass a non-string-literal to `import.meta.glob`
 - trying to escape your package via `../`
 - trying to import a pattern that doesn't start with `./` or `../`

## Drawbacks

As designed, this is not a drop-in replacement for the old system. Addons that relied on the looseness of the old system are going to need to make breaking changes to their public API to adapt to this change. I think those breaking changes are likely to be "constant cost" changes that are not expensive, even for big application, so I consider this worth it in order to get us into a more long-term-supportable position that is compatible with standard Javascript.

## Alternatives

### Globally powerful import.meta.glob

We could attempt to allow more globally-powerful `import.meta.glob`. For example, it might be possible to make patterns starting with `/` always search the current application, even when an addon is asking. This would give addon authors more of the power they're used to having, but I think it's a much riskier feature to enable across the ecosystem. I'm not convinced we could make it work at reasonable cost in even all current build systems, never mind future ones. As written, this RFC has low-risk of causing compatibility problems in the future since the feature is not allowed in addon publication format. This makes it much easier to evolve the feature over time without breaking the universe.

### Additional Vite feature space

Features from Vite's implementation that I didn't incorporate because I don't want to sign us up to reimplement them in every build system:

 - absolute imports, starting with `/`. This is not a well-defined concept in today's Ember apps because they do not have a single directory representing the app's web root.
 - `as: 'raw'` which gives you the raw source code of matching files
 - `as: 'url'` which gives you URLs to the matching files
 - named imports mode, which allows you to ask for specific names instead of whole modules
 - custom queries

None of these are necessarily bad, but they aren't strictly necessary to meet our needs and a more minimalist spec is more likely to remain stable and supported over the long term.  An app that's using Vite is free to use Vite-specific extensions if they choose to be accept that dependency. 

