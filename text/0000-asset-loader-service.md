- Start Date: 2016-07-22
- RFC PR: [emberjs/rfcs#158](https://github.com/emberjs/rfcs/pull/158)
- Ember Issue: (leave this empty)

# Summary

The Asset Loader service is an [Ember Service](http://emberjs.com/api/classes/Ember.Service.html) that is responsible for loading assets specified in an [Asset Manifest](https://github.com/emberjs/rfcs/pull/153). The API for loading these assets is Promise-based for integration with [the `getHandler` API from Router.js](https://github.com/tildeio/router.js/pull/183).

Note: It is recommended that you read the [Asset Manifest RFC](https://github.com/emberjs/rfcs/pull/153) before reading this RFC.

# Motivation

The Asset Loader service is needed to provide a standard way of loading additional assets   for an Ember application at runtime, just as the Asset Manifest specifies a standard way of describing assets that can be loaded at runtime.

In particular this will help enable lazily loaded Engines which need a standard and routing-integrated way of loading assets. While the initial use case is to support lazy-loading Engines this will allow loading of any additional resources into an Ember application, such as heavy libraries for charting.

## Goals

- Simple API for loading assets and bundles of assets.
- Prevent duplicate requests for the same asset.
- Supports the Asset Manifest specification.
- Integrates with Router.js to enable routing-based loading of assets.
- Pluggable interface for specifying how to load assets of a given type.

# Detailed Design

Initial implementation and iterations will be done in the [ember-asset-loader](https://github.com/trentmwillis/ember-asset-loader) addon. Since it is likely this functionality can be achieved entirely in user space, it is unknown whether this will be upstreamed into Ember.js proper in the future. Regardless, it should become a community-owned approach to loading assets.

## Types and Interfaces

The Asset Loader should initially be considered a private Service to allow iteration until a reliable interface has been established.

The API to be used internally (and potentially the future public API) for the service is defined as follows (assumes adherence to the interfaces defined by the Asset Manifest):

```js
interface AssetLoader {
  pushManifest(manifest: AssetManifest),
  loadAsset(asset: Asset): AssetPromise,
  loadBundle(bundle: BundleName): BundlePromise,
  defineLoader(type: String, loader: (uri: String) => Promise<T, U>)
}
```

The `AssetPromise` and `BundlePromise` types shown above are defined as:

```js
type AssetPromise = Promise<Asset, AssetLoadError>;
type BundlePromise = Promise<BundleName, BundleLoadError>;
```

Meaning that an `AssetPromise` will resolve with the information of the `Asset` it was loading while the `BundlePromise` will resolve with the name of the `Bundle` it was loading. In both cases, rejections will be handled by special types of errors which are defined as:

```js
interface LoadError extends Error {
  retryLoad(): Promise<T, U>
}

interface AssetLoadError extends LoadError {
  asset: Asset,
  retryLoad(): AssetPromise
}

interface BundleLoadError extends LoadError {
  bundle: BundleName,
  loadErrors: Array<LoadError>,
  retryLoad(): BundlePromise
}
```

The `LoadError` classes will be discussed in-depth in a later section.

## `AssetLoader` Methods

### `pushManifest`

The `pushManifest` method allows configuration of the Asset Loader service by providing an object that conforms to the Asset Manifest specification. This will allow the loader to know what assets are available to load and where to find them.

Additional invocations will add manifests into an internal data structure for later reference. When attempting to load bundles or perform a task that require referencing the manifest, the `bundles` property in all the manifest will be merged. In the case of merge conflicts, an error will be thrown to avoid inconsistent and unintuitive loading behavior.

Since the Asset Loader will be integrated with Ember itself and the Asset Manifests will be Ember CLI output, this method is the interface through which to explicitly provide the external manifest to the service.

Note: This method _must_ be called at least once prior to using `loadBundle` in order to provide the information needed to load any bundles. This should likely be done in an instance-initializer.

#### Example

```js
// instance-initializers/asset-loader-manifest.js
import manifest from 'asset-manifest';

export function initialize(instance) {
  const loader = instance.lookup('service:asset-loader');
  loader.pushManifest(manifest);
}

export default {
  name: 'asset-loader-manifest',
  initialize
};
```

### `loadAsset`

The `loadAsset` method loads a single asset into the application. It accepts an `asset` object which contains `uri` and `type` fields, both of which are strings. The `type` determines which loading method will be called internally according to the type of asset being loaded and the `uri` will be passed as an argument to said loading method to specify the actual asset to load.

`loadAsset` will return an `AssetPromise` that will resolve when the asset is successfully loaded or reject when the asset fails to load.

Subsequent calls to `loadAsset` with the same arguments will return the same `AssetPromise` object to ensure a single source of truth for determining asset state as well as to prevent duplicate requests being made for the same asset.

Note: `loadAsset` will initially support `js` and `css` type assets with additional types being supported via the `defineLoader` interface specified below.

#### Example

```js
function loadBlogVendor() {
  const asset = {
    uri: '/assets/blog/vendor-gqjszdtdmxhjuvcu.js',
    type: 'js'
  };
  const assetPromise = AssetLoader.loadAsset(asset);
  return assetPromise.then(assetLoaded, assetErrored);
}
```

### `loadBundle`

The `loadBundle` method is highly similar to the `loadAsset` method. The two primary differences are as follows.

`loadBundle` accepts a `BundleName` and not a `Bundle` itself. The name is used to look up the actual `Bundle` definition from the asset manifest. This is largely for simplicity's sake and easy cache-ability.  This could be expanded in the future to allow arbitrary `Bundle` definitions to be passed in, but there is no use case for it in the initial iteration.

Additionally, `loadBundle` returns a single `BundlePromise` that is actually the aggregation of all the `AssetPromises` from loading the assets specified in the bundle and its dependencies. This is likely to be implemented as a `Promise.allSettled`.

Thus, it is not possible to doubly load a bundle, as loading a bundle is simply loading a collection of assets. Since `loadAsset` protects against double loading we should be safe in this method as well.

#### Example

```js
function loadBlog() {
  const bundlePromise = AssetLoader.loadBundle('blog');
  return bundlePromise.then(bundleLoaded, bundleErrored);
}
```

### `defineLoader`

The `defineLoader` method allows additional or replacement loading methods to be defined for specific types of assets.

The first parameter is a string representing the type of asset this loader is responsible for (e.g., `'png'` or `'json'`). If a loader has already been defined for assets of the specified type, then that loader will be replaced with your newly defined loader.

The second parameter is the loader function. A loader function accepts a `uri` string as an argument and returns a `Promise` that represents the loading status of the asset.

#### Example

```js
// specifies a new type of loader
assetLoader.defineLoader('json', function jsonAssetLoader(uri) {
  return $.getJSON(uri);
});

// overrides an existing loader
assetLoader.defineLoader('js', loadJsThroughServiceWorker);
```

## `LoadError` Classes

The primary goals of the `LoadError` classes are:

1. Provide robust feedback in the event an asset fails to load, and
2. Provide a way to attempt recovery of the user flow

### Provide Robust Feedback

To provide meaningful feedback to developers, both of the error classes report back what they were trying to load. This means for `AssetLoadErrors` there is an `asset` property which contains the asset's information. For `BundleLoadErrors` this is a `bundle` property which contains the bundle's name.

Additionally, for `BundleLoadErrors` an `errors` property will contain all the errors that resulted in the bundle failing to load. This will give developers information as to exactly which assets and dependencies had problems while loading.

### Provide A Way To Attempt Recovery

When an asset fails we want a way to allow users to retry the load in case the failure was intermittent. However, we also want to guarantee that we aren't duplicate loading assets.

To enable both of these, the `LoadError` classes specify a `retryLoad` method that is highly similar to the `loadAsset` and `loadBundle` methods.

`retryLoad` will defer to either `loadAsset` or `loadBundle` depending on the specific type of load being retried. However, since those results are cached, the method will first evict the previous results from cache. The new load Promise will then be set as the cached value for the return of both `loadAsset`/`loadBundle` and `retryLoad`, this will guarantee we only ever have a single request for a given asset at one time.

# How We Teach This

Asset loading should be considered an advanced topic and initially should be introduced alongside Engines. That documentation can likely live in the `ember-engines` addon until the time at which all behavior has been upstreamed into Ember core.

In the future, if additional use cases are uncovered then the documentation should be a standalone section, though it should likely continue to be considered an advanced use case that does not affect new users.

# Drawbacks

This introduces another concept for advanced Ember users to understand and process. It also locks us into a new standard for loading assets which will need to be maintained.

# Alternatives

The exact semantics of the API can be bikeshedded along with parameter and return types.

The other primary design considered is to use asset loading utility functions, akin to `$.getScript()`. The primary drawback to this approach is that it is difficult to maintain the state needed to prevent duplicate loading of resources in an encapsulated manner.

# Unresolved Questions

- Should `pushManifest` return a value?
  - Could return the combined manifest, but that could lead to abuse and I don't see a use case for it.
  - Specify merge/assign behavior shallow vs. deep. Maybe throw on collision? Review again once we have the implementation.
  - Could return a value to later evict a manifest, but we would need to define how that interplays with the caching.
