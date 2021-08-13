---
Stage: Accepted
Start Date: 2021-08-13
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js, Ember CLI
RFC PR: https://github.com/emberjs/rfcs/pull/763
---

<!---
Directions for above:

Stage: Leave as is
Start Date: Fill in with today's date, YYYY-MM-DD
Release Date: Leave as is
Release Versions: Leave as is
Relevant Team(s): Fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies
RFC PR: Fill this in with the URL for the Proposal RFC PR
-->

# Asset Import Spec

## Summary

This RFC defines standard semantics for what it means to depend on files that are not Javascript or CSS, like images, fonts, and other media. It proposes that when your code needs to refer to an asset, you should import it, which will give you back a valid URL:

```js
import myImage from './hello.png';
class extends Component {
  myImage = myImage
}
```
```hbs
<img src={{this.myImage}} />
```

## Motivation

Apps and addons often need to refer to images, fonts, etc from within their JS, HBS,CSS, or HTML files. These references need to remain correct even as both the referring file and the referred-to file go through a build that might include fingerprinting and optimization.

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

And because broccoli-asset-rev exposes assumptions about the exact layout of the final build output, it's not compatible with Embroider. Embroider allows different Javascript packagers to optimize the app delivery in any spec-compliant way. For example, small images might get inlined as `data:` URIs, CSS `@import()` might get inlined or not, etc. All these concerns really need to be integrated into the Javascript packager, because asset handling affects app-wide hashing & fingerprinting.

In contrast, a pull-based design that lets code declare what assets it needs and then not worry about how those assets will get delivered is safer and easier to change in the future.

## Detailed design

Inter-file references take many forms. Here is a guide to inter-file references organized by which kind of file is doing the referring ("From Type") and which type of file it's referring to ("To Type"). Only the row containing **Proposed** is new in this RFC:

From Type | To Type | Example Syntax | Semantics
--------------------|-----------------------|--------|---
JS | JS | `import { stuff } from 'thing';`| ECMA with NodeJS resolving rules
JS | CSS | `import 'thing/style.css';` | [Embroider Spec CSS Rule](https://github.com/emberjs/rfcs/blob/master/text/0507-embroider-v2-package-format.md#css) (1)
JS| Other Asset (2) | `import url from 'thing/icon.png';`| **Proposed** URL Rule (3)
CSS | CSS | `@import("thing/style.css")` | [W3C](https://drafts.csswg.org/css-cascade/#at-import) plus NodeJS resolving rules
CSS | Other | `url("thing/icon.png");` | W3C plus NodeJS resolving rules
HTML | JS | `<script src="./thing.js"></script>` | W3C
HTML | CSS | `<link rel="stylesheet" href="./thing.css"/>` | W3C
HTML | Other | `<img src="./thing.png" />` | W3C


1. Importing a file with an explicit `.css` extension guarantees that the given CSS will be loaded into the DOM before your module executes. We do not define any exported values, this is purely for side-effect. This rule is not part of the present RFC, it was already in RFC 507.
2. "Other Asset" means this rule applies to any JS import of a path that ends in an explicit file extension that is not `.js` or `.css`, meaning:
    ```js
    function isOther(theImportedPath) {
      let extension = /\.([^.\/]+)$/.exec(theImportedPath)?.[1]?.toLowerCase();
      return extension && !['js', 'css'].includes(extension);
    }
    ```

3. Importing any "Other Asset" file gives you back a `default` export containing a valid URL to that file.

### What about references from HBS?

This RFC doesn't treat HBS separately because as far as the module system is concerned, HBS files are just JS files. They're transpiled into JS before any module traversal can happen. This RFC doesn't define any static syntax for use in templates because we should use whatever syntax is designed to go with [Strict Mode](https://github.com/emberjs/rfcs/blob/master/text/0496-handlebars-strict-mode.md). For example, in GJS you could say:

```js
import myImageURL from './my-image.png';
<template>
  <img src={{myImageURL}} />
</template>
```

Today, the recommendation should be to use the JS part of a component to import assets and pass the resulting URLs to the HBS part of the component.

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
import url0 from './thing.png';
// all the rest of this is existing low-level API from other merged RFCS:
import { setComponentTemplate } from '@ember/component';
import { precompileTemplate } from '@ember/template-compilation';
import templateOnlyComponent from '@ember/component/template-only';
export default setComponentTemplate(
  precompileTemplate('<img src={{url0}} />', { scope: () => ({ url0 }) }),
  templateOnlyComponent()
);
```

### Extensibility

There are many alternative competing ways to assign extra semantics to imports of non-JS files. Examples include:

 - importing `.graphql` files, which get preprocessed into a Javascript value, possibly with the side effect of tracking that this particular query is in use.
 - doing "CSS in JS" where an imported CSS file comes back as a Javascript value.

This RFC is not in conflict with these patterns, because they should be understood as authoring formats that can be compiled into a spec-compliant format for compatibility with the rest of the build system and ecosystem. In both the above cases, a preprocessor would rewrite these files to Javascript, and then they get completely standard handling from that point forward.

App authors are still always free to configure their build tool with this kind of preprocessing. They may need to be careful to restrict their customizations to only run on their own code. For example, if you want imports of `.png` files to have a special behavior in your app, that's possible, but you shouldn't rewrite `.png` imports in your addons because they're expecting the standard behavior defined by this RFC. On the other hand, it's fair game to configure your build tool in ways that preserve semantics. For example, configuring your build to run all imported `.png` files through a compression optimizer would be OK even for images imported by addons.

Addon authors should ideally run any preprocessing before publishing. If they want apps to also do some preprocessing, they should provide explicit instructions & utilities for app authors to configure preprocessing in their particular build tool. They should not try to automatically rewrite the app's build system configuration, because that way lies madness and ecosystem-wide lock-in.

### Asset file location compatibility

Under the Embroider v2 spec, there are no implementation restrictions on where you could put assets in your package. You can access them via relative imports regardless, and consumers of your package can access them under your package name, following Node's standard rules (including the `exports` key in package.json).

But in today's v1 addons and apps, there are some compatibility issues with assets and their paths. Your module namespace is rooted not at the root of your package, but at the `/app` folder. So importing "your-app/logo.png" logically means "/app/logo.png". You cannot address anything in the traditional "/public" folder.

This is actually fine: anything in `/public` is still "push-based", meaning it will end up in your final build regardless of whether it's consumed by anything, and we really *guarantee* that it will be there. Whereas any assets you import following this new spec are only included when they're used, and their actual final URL is not guaranteed to take any particular form (they may even be `data:` URIs with no real file in the build output). So we don't want these new-style assets to be under `/public`.

My recommendation is that any asset file that is used by a single component should be co-located beside that component. And assets that are shared throughout the app can go into directories like `/app/images`, etc. As long as you keep everything under `/app` it will all work as expected.

### Build errors

A benefit of this design is that trying to import an asset that does not exist is a build error. You cannot fail to notice if you accidentally delete an asset that is still in use somewhere in your app.

### Upgrade path

This proposal can be implemented with relatively small changes to ember-auto-import. Once you upgrade to a version of ember-auto-import with these features, you can begin importing assets and getting valid URLs for them. None of this breaks traditional asset handling including broccoli-asset-rev, so a gradual conversion is possible.

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
 - build systems could implement `import "./some.css"` by generating Javascript that inserts an inline stylesheet or a link tag, or by inserting a link tag directly into the HTML.

The exceptions where you can guarantee control over the URLs of assets are:

 - Files in `/public` are guaranteed to get URLs you can control (and manipulate with the traditional `fingerprint` options).
 - The traditional `index.html`, `app.js` and `vendor.js` files (plus their test equivalents) continue to get URLs you can control in the usual way.

## How we teach this

ember-auto-import is already part of the default app blueprint, so newly-generated apps would gain these capabilities with no changes to their configuration or dependencies. That means tutorials & guides can show examples of, for example, how to include an image in a component.

The [Assets and Dependencies docs](https://cli.emberjs.com/release/basic-use/assets-and-dependencies/) would change to show an example of importing a co-located asset directly into your code, in addition to mentioning that `/public` can be used for standalone assets that need their own URL.

The [Asset Compilation docs](https://cli.emberjs.com/release/advanced-use/asset-compilation/) would change to explain that `/public` is for files that must live at particular URLs in your final app, whereas if you just need to make sure an asset is available to your code, you should keep it within `/app` and import it where it's used. The fingerprinting section can stay as it is for now, though people should be aware that it doesn't apply anymore under embroider, so as we approach enabling embroider by default there will be a future RFC that causes that whole section to go away.

## Drawbacks

One footgun is that the module namespace of today's Ember apps doesn't match the on-disk package layout. It would be natural to think that `/app/app.js` could `import "../public/thing.png"`, but this doesn't work because nothing above `/app` is addressable as a module. We should think about ways to provide helpful feedback if people try this. A lint rule against relative imports that reach above "/app" would be one possibility.

## Alternatives

We could choose to let apps each configure asset importing in their own way. This is easier to design because we avoid responsibility, but it pushes more complexity onto app authors and it makes life harder for addon authors who want to have some supported thing they can rely on across all apps.

It would be possible to replace broccoli-asset-rev's unsafe reference rewriting with a runtime API, like a service plus helpers. This requires no build-time features and it would avoid assigning new semantics to ECMA `import`, but it wouldn't have the ability to analyze the asset dependency graph at build time.

We could address the confusing module namespacing of the `/app` folder by making apps configure the `exports` key in package.json. This would also give them the option of putting importable assets in other locations that are not under `/app`, by extending the `exports` configuration. And while it is relatively simple to make ember-auto-import and Embroider respect `exports`, I don't think it would be easy to make the classic build respect `exports`. So that could introduce new sources of confusion.
