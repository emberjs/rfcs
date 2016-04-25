- Start Date: 2016-04-13
- RFC PR: (leave this empty)
- ember-cli Issue: (leave this empty)

# Summary

Provide an overview of the parts required to enable different lazy-loading scenarios for Ember Applications.

# Motivation

Lazy loading helps us optimize boot time, by minimizing the amount of code that is loaded and evaluated during boot.
This allows apps to grow horizontally, support different user roles or take more dependencies without
necessarily increasing the app's payload or the costs associated with it.


# Detailed design

Lazy loading is complex and applies to multiple scenarios. In order to simplify its implementation I'm breaking it
in different, independent parts, all of those could be delivered incrementally and still provide value. We need to
build different assets, provide a standard way to load and track assets and support hooks to lazy load assets from routes
and components. It is also important to consider asset dependencies.

## Building

We need to change the way we build our assets. By default Ember CLI bundles everything in two app and vendor assets for
CSS and JS. In order to be able to lazy-load, we need to allow for customizations to how we build. This is partially
possible today, but further enhancements are required.

### Building multiple CSS:

Today we have two ways of building multiple CSS assets. For vendor static assets (from `vendor/` or `bower_components`)
we can import them to a different `outputFile` as specified on
[RFC#28](https://github.com/ember-cli/rfcs/pull/28) and supported since ember-cli 2.4.

```
app.import(app.bowerDirectory + '/bootstrap/dist/css/bootstrap.css', {outputFile: 'vendor2.css'});
```

For files that need a pre-processor we can use `outputPaths`. In the example below, `styles/deferred-styles.scss` is
simply a file with references to other CSS or SASS files. This works for `vendor` and `app` styles.

```
let app = new EmberApp(defaults, {
  outputPaths: {
    app: {
      css: {
        'app': '/assets/css/app.css',
        'deferred-styles': '/assets/css/deferred-styles.css',
      }
    }
  }
});
```

### Building multiple vendor JS assets from static files (e.g. bower/vendor):

For vendor static assets (from `vendor/` or `bower_components`) we can import them to a different `outputFile`
as specified on [RFC#28](https://github.com/ember-cli/rfcs/pull/28) and supported since ember-cli 2.4.


```
app.import('vendor/dependency-1.js', { outputFile: 'assets/alternate-vendor.js'});
```

### Building multiple vendor JS assets from addons

There is an [RFC](https://github.com/ember-cli/rfcs/pull/52) for this, but the concept is similar to the above, but
extended for addons.

### Building multiple JS assets for app code

This isn't supported out of the box today, but it's possible to do. There are two example,
[ember-cli-bundle-loader](https://github.com/MiguelMadero/ember-cli-bundle-loader) and
[ember-lazy-loader](https://github.com/duizendnegen/ember-cli-lazy-load). The first one relies on a different file structure 
and uses multiple ember-apps to build each asset. The latter relies on configuration to specify which app files will go to each
asset.

This RFC is limited to provide an overview of what is required, so this topic requires a bit more discusion on a
separate RFC. TODO: create the separate RFC. Engines can provide a nice default, where we can have a convention of
one asset per engine when lazyLoading is enabled, which can be overriden to allow for one or more engines per file.

## Loading assets

[ember-cli-bundle-loader](https://github.com/MiguelMadero/ember-cli-bundle-loader) provides a
[service](https://github.com/MiguelMadero/ember-cli-bundle-loader/blob/master/addon/services/lazy-loader.js)
to dynamically load CSS and JS. This service will be extracted to its own add-on and extended to support the
asset metadata described below.

## Lazy loading from components

The service can be used by components when they need to load vendor JS or CSS assets. We have an example of this at
Zenefits, where some of our components that depend on third-party libraries lazy-load them. In this particular case,
we load the code on init and certain functions of the component are only available after the library is loaded.
All of the code is removed to focus on the lazy-loading parts.

```
// app/components/z-table.js
export default Ember.Component.extend({
  lazyLoader: Ember.inject.service()
  init() {
    // This call is idempotent and the path will be mapped to a finger printed veresion
  	this.get('lazyLoader').load('hands-on-table.js');
  }
});

// app/components/z-table.hbs
{{#if get(lazyLoader.loadedLibraries "hands-on-table.js"}}
  {{!-- display other functionality that depends on the lazy-loaded lib --}}
{{/if}}

// ember-cli-build.js
let app = new EmberApp(defaults, {});
app.import('bower_components/hands-on-table/dist/hands-on-table.js', {outputFile: 'hands-on-table.js');
```


## Lazy Loading from routes

For app code, we likely want an integration with the router, which can rely on this service to do the actual loading.

When splitting the app's JS and CSS into multiple assets related to different sectionsof an application, we want to load the code dynamically. This today poses additional challenges because the handler (route) needs to exist to initiate a
transition, but that route class doesn't exist until we load the new code. Also link-to depends on the route information
in order to `serialize` the model information to create the URL. That means that even before attempting to
transition, the route needs to be available. The latter problem was solved with the
[serialize changes](https://github.com/emberjs/rfcs/pull/120). To add the ability to load code and lazily load routes,
we need a new hook in Ember's router or router.js, ongoing work for this is being done at the moment to make the
`getHandler` function asynchronous. Once that is done, we can call the loading-service similar to what we do in
components and resolve with the name of the asset to be loaded based on route metadata. The following code is
just an example and assumes we have the route metadata and the loadingService available.
A final implementation of how this hook will work is out of the scope of this RFC.

```
Router.getHandler = function (routeName) {
  let assetName = routerMetadata.getAssetFor(routeName);
  if (lazyLoaderService.loadedLibraries["assetName"]) {
    return this._super(...arguments);
  }
  return lazyLoadingService.load(assetName).then(()=> {
    this._super(...arguments);
  });
};
```

### Route metadata

The code above assumes that we have the routerMetadata available, the definition of this needs further discussion 
outside of the scope of this RFC. For the purpose of this overview we simply assume that we have a way to map between routeNames and assetNames.

## Engines

Engines can simply rely on all of the parts above and provide nice conventions.

## Dependencies and concatenation

Sometimes an asset has a dependency on another asset. For example,
[ember-cli-bundle-loader](https://github.com/MiguelMadero/ember-cli-bundle-loader/blob/master/tests/dummy/config/bundles.js)
allows you to specify dependencies bettween what they call `bundles`, the `lazyLoaderService` can make sure that the
dependencies are loaded in parallel, but evaluated in the correct order and not loaded more than once.

Another optimization is to concatenate dependencies into a single asset to avoid multiple requests. Today we can solve
it during the build process by using the same `outputFile`, for engines and app code we might need to override some of
the build defaults to oveirrde the output file.

## Pre-requisites

[Serialize RFC](https://github.com/emberjs/rfcs/pull/120)

# Drawbacks

TODO: fill this in

* Without proper static analysis, it exposes the consumers to errors if there are dependencies in the order things are loaded
and dependencies between assets aren't correctly specified. 
* The way to build different assets is inconsistent. 


# Alternatives

This section needs further discussion. 

TODO: fill this in

# Unresolved questions

TODO: fill this in, but the whole RFC is already full of unresolved questions, some of which deserve their own RFC
