- Start Date: 2016-11-21
- RFC PR: [#80](https://github.com/ember-cli/rfcs/pull/80)

# Summary

This RFC attempts to expose a public API in `ember-cli` to allow other platforms/infrastructure to serve the base page (index.html) and other assets from the `tmp` directory in their own custom way. This is only for development as this will only be used with `ember serve`. Currently `ember serve` serves files from the tmp directory (which is built as part of the build process) using the `broccoli-middleware`. This middleware in addition to serving the files also sets the correct headers for the assets it is serving. This RFC aims to split the work of setting the header and serving the files into two different addons such that any other infrastructure can easily create a middleware to serve assets using its own logic.

# Motivation

FastBoot and other infrastructure (for example the infrastructure at LinkedIn to serve the base page) does not require the index.html to be served from the disk directly. FastBoot requires to serve the index.html after it has appended the serialized template for the current request. It therefore requires to do some runtime replacements in the index.html before it can be served to the client. At LinkedIn, we stream the index.html in chunks for performance reasons and require to do some string replacements in index.html on per request basis.

During development, this requires us to create our own  express middleware via `serverMiddleware` which should run before the `serve-files` middleware. It also requires us to almost copy paste the headers that are set by `broccoli-middleware`. In addition to the above, the ability to be able to serve from `tmp` directory allows FastBoot and other infrastructure to correctly serve the assets from the directory pointing to the current build. Currently (with using their own middleware) FastBoot serves assets from the `dist` directory which is not the correct behavior.

In order to mitigate the need to diverge into another middleware which behaves almost same as `broccoli-middleware`, this RFC proposes to split the work of setting the headers and serving the files via `broccoli-middleware` and expose a public API that will allow an addon to define how it wants to serve the assets. It will be a low level public API that will be invoked by certain addons.

# Detailed design

Currently the [`serve-files`](https://github.com/ember-cli/ember-cli/blob/375f3a32f4564465d2eccc3815cb61b570ce29f0/lib/tasks/server/middleware/serve-files/index.js#L18) addon defines how it will serve the incoming asset requests. It invokes the `broccoli-middleware` which is responsible for three sets of things:

1. Setting the response headers

2. serving the files and ending the response

3. Shows an error template if it is a build error

## Public API
This RFC proposes expose two new in-repo addons in `ember-cli` which will now split the above work and remove the `serve-files` addon:

1. `ember-cli:broccoli:watcher`: This addon will contain the middleware which will be responsible for making sure the build is done and will set the response headers that `broccoli-middleware` is doing today. After setting the response headers, it will call the next middleware in the chain. In addition, if the build results in an error, it will show the error template and not terminate the response.

2. `ember-cli:broccoli:serve-files`: This addon will always run *after* `ember-cli:broccoli:watcher` addon. It will contain a middleware that will be responsible for serving the files from the filesystem and ending the response.

For any infrastructure that needs to serve the assets in its own way will be create an addon that will be injected between the above two addons. It will use the `serverMiddleware` public hook to provide its own middleware. Specifically the custom addon should run *before* `ember-cli:broccoli:serve-files` so that it can either override any response headers or can serve the files using its own logic and end the response. This will ensure that when the build is successful `ember-cli:broccoli:watcher` can call the correct next middleware in the chain.

## Implementation Details
In order for the above API to be exposed, we need to drop the `serve-files` addon in `ember-cli`, refactor `broccoli-middleware` and create the two new addons.

### Refactor `broccoli-middleware` to expose additional middlewares
*Note*: This refactor section is only for making the reader understand how the integration is meant to work in `ember-cli`. This is not going to be `ember-cli` public API.

`broccoli-middleware` is currently responsible for setting the response headers and serving the files. It is a middleware that does these two tasks. It doesn't expose a proper middleware API to do the two tasks differently. We would like to refactor `broccoli-middleware` such that it exposes two additional middlewares:
- `forWatcher(watcher)`:
```javascript
  /**
   * Function responsible for setting the response headers or creating the build error template
   *
   * @param {Object} watcher ember-cli watcher
   * @return {Function} middleware function
   */
   forWatcher: function(watcher) {
     var outputPath = watcher.builder.outputPath;
     ...
     return function middleware(request, response, next) {
       watcher.then(function() {
         // mostly all of this https://github.com/ember-cli/broccoli-middleware/blob/master/lib/middleware.js#L96
         request.headers['x-broccoli'] = {
           outputPath: outputPath
         };
         next();
       }, function(buildError) {
         // mostly this: https://github.com/ember-cli/broccoli-middleware/blob/master/lib/middleware.js#L121
       })
     }

   }
```

- `serveFiles()`:
```javascript
 /**
  * This function will be responsible for serving the files from the filesystem
  *
  * @param {HTTP.Request} request
  * @param {HTTP.Response} response
  * @param {Function} next
  */
  serveFiles: function() {
    return function(req, resp, next) {
      // get the output path from from the request headers
      // most of `broccoli-middleware` https://github.com/ember-cli/broccoli-middleware/blob/master/lib/middleware.js#L115
    }
  }
```

### Create `ember-cli:broccoli:watcher` addon
The current `serve-files` addon invokes the `broccoli-middleware` and delegates the task to this middleware to serve the files and set the headers. We would like to change that and instead this new in-repo addon `ember-cli:broccoli:watcher` should only call `setResponseHeaders` function from `broccoli-middleware`. The `serverMiddleware` function of this [addon](https://github.com/ember-cli/ember-cli/blob/375f3a32f4564465d2eccc3815cb61b570ce29f0/lib/tasks/server/middleware/serve-files/index.js#L18) will now look as follows:
```javascript
  ServeFilesAddon.prototype.serverMiddleware = function(options) {
    var app = options.app;
    var watcher = options.options.watcher;
    var broccoliMiddleware = require('broccoli-middleware');

    app.use(function(req, resp, next) {
      // copy over this: https://github.com/ember-cli/ember-cli/blob/375f3a32f4564465d2eccc3815cb61b570ce29f0/lib/tasks/server/middleware/serve-files/index.js#L33
      if (options.options.middleware) {
        // call the middleware that is provided for testemMiddleware
      } else {
        var watcherMiddleware = broccoliMiddleware.forWatcher(watcher);

        watcherMiddleware(req, resp, function(err) {
          if (err) {
            // log error
          }
          next(err);
        });
      }
    });
  }
```

As seen above `ember-cli:broccoli:watcher` will only be responsible for setting the headers and calling the the next middleware which will serve the files.

### Create `ember-cli:broccoli:serve-files` addon
We will create a new in-repo addon called as `ember-cli:broccoli:serve-files` which will be responsible for serving the the files. This addon will run *after* `ember-cli:broccoli:watcher` addon.

This function will be responsible for serving the incoming asset request from the filesystem. It will use the `serverMiddleware` API to serve the files using `broccoli-middleware`.
```javascript
  BroccoliServeFilesAddon.prototype.serverMiddleware = function(options) {
    var broccoliMiddleware = require('broccoli-middleware');
    var outputPath = options.watcher.builder.outputPath;
    var autoIndex = false;

    var options = { outputPath, autoIndex };
    app.use(function(req, resp, next) {
      var serveFileMiddlware = broccoliMiddleware.serveFiles();
      serveFileMiddlware(req, resp, function(err) {
        next(err);
      })
    });
  }
```

In order for FastBoot to be able to serve the assets using its own logic, it will specific that it run *before* `ember-cli:broccoli:serve-files` addon so that it can serve the assets. In this way, FastBoot will be able to inject itself into the correct order and be able to serve assets from the `tmp` directory.

### Drop `serve-files` addon in ember-cli
Since the work that `serve-files` does today is now split into two new in-repo addons, `serve-files` addon doesn't need to be present any longer. It is not exposing any public API or functionality that users may be using today and therefore can be dropped.

# How We Teach This

We will need to update the `ember-cli` website with this new in-repo addon and specify the above usecase with an example.

# Drawbacks

The only drawback is addon authors wanting to serve assets using their own logic, will need to know the correct order of middleware execution. Moreover, if someone has forked `ember-cli` to hack `serve-files` addon logic, it will be a breaking change for them.

# Alternatives

N/A

# Unresolved questions
- [ ] Should this addons be better named?
