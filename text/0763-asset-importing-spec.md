---
stage: ready-for-release
start-date: 2021-08-13T00:00:00.000Z
release-date:
release-versions:
teams:
  - cli
  - framework
  - learning
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/763'
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



# Asset Import Spec

## Summary

This RFC defines standard semantics for what it means to depend on files that are not JavaScript or CSS, like images, fonts, and other media. It proposes that when your code needs to refer to an asset, you use `import.meta.resolve` which returns a string that can be used to fetch the asset.

```js
class extends Component {
  myImage = import.meta.resolve('./hello.png');
}
```
```hbs
<img src={{this.myImage}} />
```

## Motivation

Apps and addons often need to refer to images, fonts, etc from within their JS, HBS, CSS, or HTML files. These references need to remain correct even as both the referring file and the referred-to file go through a build that might include fingerprinting and optimization.

Today, the ecosystem mostly relies on [broccoli-asset-rev](https://github.com/ember-cli/broccoli-asset-rev). But broccoli-asset-rev has some major architectural problems and it doesn't take advantage of the newer capabilities we have in ember-auto-import and embroider.

The biggest problem with broccoli-asset-rev is that it cannot achieve full correctness because it tries to detect inter-file references within arbitrary source code that is not statically analyzable. It's easy to accidentally refactor working code into code that works in development and test and then fails in production! For example, assuming broccoli-asset-rev is configured to fingerprint PNG assets, this refactor will look correct until you get into production and then have broken images:

```diff
class extends Component {
-  iconURL = "/icons/happy.png"
+  get iconURL() {
+    return `/icons/${this.args.emotion}.png`
+  }
}
```

The regular expressions that broccoli-asset-rev uses to do reference rewriting are effectively un-editable at this point. Any possible change you'd make to them breaks somebody's app.

Another problem with broccoli-asset-rev is that it's "push based", meaning it handles files because they exist, rather than handling files because somebody is actually consuming them. It can't detect unused assets and it can't detect missing assets at build time. It exposes the underlying implementation of how assets get delivered, making it hard to change later without breaking app code.

And because broccoli-asset-rev exposes assumptions about the exact layout of the final build output, it's not compatible with Embroider. Embroider allows different JavaScript packagers to optimize the app delivery in any spec-compliant way. For example, small images might get inlined as `data:` URIs, CSS `@import()` might get inlined or not, etc. All these concerns really need to be integrated into the JavaScript packager, because asset handling affects app-wide hashing & fingerprinting.

In contrast, a pull-based design that lets code declare what assets it needs and then not worry about how those assets will get delivered is safer and easier to change in the future.

## Detailed design

Inter-file references take many forms. Here is a guide to inter-file references organized by which kind of file is doing the referring ("From Type") and which type of file it's referring to ("To Type"). Only the row containing **Proposed** is new in this RFC:

From Type | To Type | Example Syntax | Semantics
--------------------|-----------------------|--------|---
JS | JS | `import { stuff } from 'thing';`| ECMA with NodeJS resolving rules
JS | CSS | `import 'thing/style.css';` | [Embroider Spec CSS Rule](https://github.com/emberjs/rfcs/blob/master/text/0507-embroider-v2-package-format.md#css) (1)
JS| Any Asset | `import.meta.resolve('thing/icon.png');`| **Proposed** URL Rule (2)
CSS | CSS | `@import("thing/style.css")` | [W3C](https://drafts.csswg.org/css-cascade/#at-import) plus NodeJS resolving rules
CSS | Other | `url("thing/icon.png");` | W3C plus NodeJS resolving rules
HTML | JS | `<script src="./thing.js"></script>` | W3C
HTML | CSS | `<link rel="stylesheet" href="./thing.css"/>` | W3C
HTML | Other | `<img src="./thing.png" />` | W3C


1. Importing a file with an explicit `.css` extension guarantees that the given CSS will be loaded into the DOM before your module executes. We do not define any exported values, this is purely for side-effect. This rule is not part of the present RFC, it was already in [RFC 507](https://github.com/emberjs/rfcs/blob/master/text/0507-embroider-v2-package-format.md).

2. Calling `import.meta.resolve` with the path to any file gives you back a string containing a valid URL for that file.

### `import.meta.resolve(moduleName)`

`import.meta.resolve(moduleName)`[^1] is a function built-in to browsers that takes a `moduleName` and "returns a string corresponding to the path that would be imported if the argument were passed to import()." [^2]

The build system will handle uses of `import.meta.resolve` when called with assets other than `.js` or .css` (see above).

The argument `moduleName` supports a subset of dynamicism as defined [here](https://github.com/emberjs/rfcs/blob/master/text/0507-embroider-v2-package-format.md#supported-subset-of-dynamic-import-syntax).

The argument `moduleName` must explicitly include the extension.

For example, the following is valid as the build system can determine the files that possibly match the interpolation:

```js
class extends Component {
  get iconURL() {
    return import.meta.resolve(`../icons/${this.args.locale}.png`)
  }
}
```

Calling `import.meta.resolve` with an asset that does not exist will cause a build error. You cannot fail to notice if you accidentally delete an asset that is still in use somewhere in your app.

#### Isn't it risky to "take over" a browser API?

No. This [original proposal for this feature to WHATWG](https://github.com/whatwg/html/issues/3871#issue-346547968) envisioned build tools rewriting the expressions to provide asset optimizations. 

The built-in function can only resolve modules relative to the active script; by using Ember's build tools developers have no guarantee of where modules build to and the feature effectively cannot be used without those build tools understanding `import.meta.resolve`.

[^1]: https://html.spec.whatwg.org/#integration-with-the-javascript-module-system
[^2]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import.meta/resolve

### What about references from HBS?

This RFC doesn't treat HBS separately because as far as the module system is concerned, HBS files are just JS files. They're transpiled into JS before any module traversal can happen. This RFC doesn't define any static syntax for use in templates because we should use whatever syntax is designed to go with [Strict Mode](https://github.com/emberjs/rfcs/blob/master/text/0496-handlebars-strict-mode.md). For example, in GJS you could say:

```js
const myImageURL = import.meta.resolve('./my-image.png');
<template>
  <img src={{myImageURL}} />
</template>
```

Today, the recommendation should be to use the JS part of a component to obtain the reference to assets and pass the resulting URLs to the HBS part of the component.

Using only Ember public API, it would also be possible to write an optional preprocessor that allows you to experiment with any authoring format you like for accessing assets from HBS. For example, you could rewrite this:

```hbs
{{!-- for illustration purposes only, this
      hypothetical "import-asset" helper is
      not being proposed by this RFC, it's
      just something you could layer on top. --}}
<img src={{import-asset "./thing.png"}} />
```

To this:
```js
// this line is the only thing in this file that is newly proposed by this RFC
const url0 = import.meta.resolve('./thing.png');

// all the rest of this is existing low-level API from other merged RFCS:
import { setComponentTemplate } from '@ember/component';
import { precompileTemplate } from '@ember/template-compilation';
import templateOnlyComponent from '@ember/component/template-only';
export default setComponentTemplate(
  precompileTemplate('<img src={{url0}} />', { scope: () => ({ url0 }) }),
  templateOnlyComponent()
);
```

### Addons

This feature provides the capability for addons to reference assets relative to their own modules.

Addon authors should ideally run any preprocessing before publishing. If they want apps to also do some preprocessing, they should provide explicit instructions & utilities for app authors to configure preprocessing in their particular build tool. They should not try to automatically rewrite the app's build system configuration, because that way lies madness and ecosystem-wide lock-in.

### Asset file location compatibility

Under the Embroider v2 spec, there are no implementation restrictions on where you could put assets in your package. You can access them via relative imports regardless, and consumers of your package can access them under your package name, following Node's standard rules (including the `exports` key in package.json).

But in today's v1 addons and apps, there are some compatibility issues with assets and their paths. Your module namespace is rooted not at the root of your package, but at the `/app` folder. So importing "your-app/logo.png" logically means "/app/logo.png". You cannot address anything in the traditional "/public" folder.

This is actually fine: anything in `/public` is still "push-based", meaning it will end up in your final build regardless of whether it's consumed by anything, and we really *guarantee* that it will be there. Whereas any assets you import following this new spec are only included when they're used, and their actual final URL is not guaranteed to take any particular form (they may even be `data:` URIs with no real file in the build output). So we don't want these new-style assets to be under `/public`.

My recommendation is that any asset file that is used by a single component should be co-located beside that component. And assets that are shared throughout the app can go into directories like `/app/images`, etc. As long as you keep everything under `/app` it will all work as expected.

### Upgrade path

This proposal can be implemented with relatively small changes to ember-auto-import. Once you upgrade to a version of ember-auto-import with these features, you can begin using `import.meta.resolve` and getting valid URLs for them. None of this breaks traditional asset handling including broccoli-asset-rev, so a gradual conversion is possible.

This RFC does not suggest removing broccoli-asset-rev from the app blueprint, because it would still be responsible for fingerprinting all the classic build output files (app.js, vendor.js, etc). Since these files are not referenced from arbitrary code, the problematic aspects of broccoli-asset-rev don't apply to them.

### Engines

Engines have some complicated interactions with asset handling today. Under the current system the URL of each asset is exposed as public API, which means that assets from different packages can potentially fight over namespace. So extra care is needed to keep them apart and give them each a view of their own assets that is namespaced properly into the app-wide namespace of URLs.

This problem doesn't exist for assets under this new proposal, because assets' precise runtime URLs are not public API. In fact, the build system would be free to notice if identical assets were included by more than one engine and deduplicate them as desired, without ever allowing an accidental collision.

### Deliberately undefined behavior

We deliberately do not define how inter-file references will be implemented. All we guarantee is that we will preserve inter-file reference semantics.

Examples:
  - build systems are free to replace any asset URL with a `data:` URI or even a blob at runtime.
  - build systems are free to rewrite CSS `@import` to inline the contents (which happens to be today's default behavior)
  - build systems can rewrite graphs of ES modules to single ES modules with the same semantics.

The exceptions where you can guarantee control over the URLs of assets are:

  - Files in `/public` are guaranteed to get URLs you can control (and manipulate with the traditional `fingerprint` options).
  - The traditional `index.html`, `app.js` and `vendor.js` files (plus their test equivalents) continue to get URLs you can control in the usual way.

## How we teach this

ember-auto-import is already part of the default app blueprint, so newly-generated apps would gain these capabilities with no changes to their configuration or dependencies. That means tutorials & guides can show examples of, for example, how to include an image in a component.

The [Assets and Dependencies docs](https://cli.emberjs.com/release/basic-use/assets-and-dependencies/) would change to show an example of using `import.meta.resolve` for an asset, in addition to mentioning that `/public` can be used for standalone assets that need their own URL.

The [Asset Compilation docs](https://cli.emberjs.com/release/advanced-use/asset-compilation/) would change to explain that `/public` is for files that must live at particular URLs in your final app, whereas if you just need to make sure an asset is available to your code, you should keep it within `/app` and reference it with `import.meta.resolve` where it's used. The fingerprinting section can stay as it is for now, though people should be aware that it doesn't apply anymore under embroider, so as we approach enabling embroider by default there will be a future RFC that causes that whole section to go away.

## Drawbacks

One footgun is that the module namespace of today's Ember apps doesn't match the on-disk package layout. It would be natural to think that `/app/app.js` could `import.meta.resolve('../public/thing.png')`, but this doesn't work because nothing above `/app` is addressable as a module. We should think about ways to provide helpful feedback if people try this. A lint rule against relative paths passed to `import.meta.resolve` that reach above "/app" would be one possibility.

## Alternatives

We could choose to let apps each configure asset importing in their own way. This is easier to design because we avoid responsibility, but it pushes more complexity onto app authors and it makes life harder for addon authors who want to have some supported thing they can rely on across all apps.

It would be possible to replace broccoli-asset-rev's unsafe reference rewriting with a runtime API, like a service plus helpers. This requires no build-time features and it would avoid assigning new semantics to ECMA `import`, but it wouldn't have the ability to analyze the asset dependency graph at build time.

We could address the confusing module namespacing of the `/app` folder by making apps configure the `exports` key in package.json. This would also give them the option of putting importable assets in other locations that are not under `/app`, by extending the `exports` configuration. And while it is relatively simple to make ember-auto-import and Embroider respect `exports`, I don't think it would be easy to make the classic build respect `exports`. So that could introduce new sources of confusion.
