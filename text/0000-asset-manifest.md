- Start Date: 2016-07-17
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

The Asset Manifest is a JSON specification to describe assets and bundles of assets that can be loaded into an Ember application asynchronously at runtime.

This is a corollary to a forthcoming Asset Loader service which will handle asynchronous loading of application assets.

# Motivation

The primary motivation for defining an Ember Asset Manifest specification is to enable [lazy-loading of Engines](https://github.com/emberjs/rfcs/blob/master/text/0010-engines.md#lazy-loading-manifests). Beyond that it can also be used for any piece-wise loading strategy of assets into an Ember application at runtime.

## Goals

- Define an interface for specifying where to load an asset needed for an application at runtime.
- Allow assets to be fingerprinted for caching reasons.
- Support multiple inclusion of the same asset without duplicating the load.
- Allow compositional loading of assets for future application segmentation.

## Secondary / Related Goals

- Keep the API of the forthcoming Asset Loader service simple.
- Enable the prevention of duplicate requests for assets.

# Detailed Design

The asset manifest will exist as the `manifest` field within the existing `config/environment` module to prevent creation of additional module definitions and to keep information centrally located.

At the core of the manifest is the idea of a "bundle" which is simply a collection of assets intended to be loaded together. Each bundle is associated with a "name" that represents the field it is under in `manifest.bundles`. Even though `bundles` will be the only field under `manifest` in the first iteration, having the additional object layer will allow us to expand the manifest to include additional information in the future.

Since we can access the package name of an Engine at runtime, regardless of aliasing, we should always load an Engine's assets according to its package name and not its mounted path. This will make it easier to avoid duplicate loading of assets and ensures we can support multiple inclusion of the same engine.

Each bundle will have two primary fields: `assets` and `dependencies`.

`assets` is an array of asset objects, which specify a `uri` and `type` for an asset to be loaded. The asset objects are sorted according to load order. This is important for assets such as `vendor.js` which need to be loaded before other assets like `app.js`.

The `asset` interface could be expanded in the future to include additional metadata information if needed, such as if two assets can be downloaded in parallel with each other.

`dependencies` is an array of names of additional bundles whose resources also need to be loaded in order for the bundle to work. While this probably won't be leveraged in the initial Lazy-Loading Engine work, it provides the ability to have more fine-grained control of asset loading in the future while retaining existing benefits.

Given this design, the Asset Loader API should then be based around the idea of "bundles" and "assets" which should keep the API simple (e.g., `loadBundle`, `loadAsset`, etc.).

## Asset & Bundle Interfaces

The above design can be expressed through a couple interfaces as follows:

```ts
type BundleName = string;

interface Asset {
  type: String,
  uri: String
}

interface Bundle {
  assets: Array<Asset>,
  dependencies?: Array<BundleName>
}

interface AssetManifest {
  bundles: Map<BundleName, Bundle>
}
```

Note: `Map` in the above can be implemented as a standard JS object.

## Asset URIs

Asset URIs will be constructed similarly to the asset URIs you find in your `index.html` as most assets will be inserted into the DOM in a similar fashion as those that are available upon page load.

The format of the asset URIs will be determined by the build process. They will, for example, take into account the `rootURL` of the host application. Additionally, tools such as `broccoli-asset-rev` will be assessed to ensure they have proper ability to modify the generated URIs for fingerprinting.

The following examples of URIs should not be taken as a set standard.

## Example

```json
// config/environment
{
  "manifest": {
    "bundles": {
      "blog": {
        "assets": [
          {
            "uri": "/assets/blog/vendor-gqjszdtdmxhjuvcu.js",
            "type": "js"
          },
          {
            "uri": "/assets/blog/engine-n92evcv0clnc1hi7.js",
            "type": "js"
          },
          {
            "uri": "/assets/blog/vendor-pdkli3mxhsfqn695.css",
            "type": "css"
          },
          {
            "uri": "/assets/blog/engine-7mykz5k5v1ljnw3f.css",
            "type": "css"
          }
        ],
        "dependencies": [
          "shared-components"
        ]
      },

      "shared-components": {
        "assets": [
          {
            "uri": "/assets/shared-components/vendor-xw6m3aj4u4eyfxt1.js",
            "type": "js"
          },
          {
            "uri": "/assets/shared-components/vendor-ivxyhmmkis3wobht.css",
            "type": "css"
          }
        ]
      }
    }
  }
}
```

# How We Teach This

The Asset Manifest should be generated at build time by Ember-CLI and consumed automatically by the Asset Loader service during runtime. Due to this, it is likely that many developers will never need to interact with the asset manifest.

However, the core concepts of defining assets and bundles and how they are loaded should be introduced and taught alongside Engines as they are the first and primary use case for this construct. This should only affect advanced users that are exploring the possibility of using lazily loaded Engines.

# Drawbacks

- Adds an additional element to consider in an Ember Application's build and runtime for advanced use cases

# Alternatives

## [Multiple Meta Modules](https://github.com/ember-cli/ember-cli/pull/5582https://github.com/ember-cli/ember-cli/pull/5582)

An alternative to the above approach is to allow each bundle to be loaded as part of the config module from each engine, a separate "meta module" if you will.  These modules would then need to be assembled by a config loader.

In this approach the primary app manifest could look like:

```json
{
  "manifest": {
    "configs": [
      "blog",
      "chat"
    ],

    "bundles": {
      "shared-components": {
        "assets": [
          {
            "uri": "/assets/shared-components/vendor-xw6m3aj4u4eyfxt1.js",
            "type": "js"
          },
          {
            "uri": "/assets/shared-components/vendor-ivxyhmmkis3wobht.css",
            "type": "css"
          }
        ]
      }
    }
  }
}
```

The `configs` field will represent additional configs that need to be integrated into the primary config. These will be looked up according to the standard convention of `<name>/environment/config`.

Each config will still have the possibility of a `bundles` field for those bundles that are not associated with a config. This is primarily for a future state where we might enable piecewise loading of assets.

The primary benefit to this approach is that the asset information for each Engine will be co-located with the rest of its configuration information.

# Unresolved Questions

- Single Vs. Multiple Configs