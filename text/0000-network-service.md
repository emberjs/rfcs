- Start Date: 2015-09-21
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Introduces a standard way of fetching data over HTTP in Ember by way of
a built-in network service.

# Motivation

The motivations for introducing a built-in network service are two-fold:

1. New users can fetch data over HTTP performantly out-of-the-box.
2. It provides a standard way of fetching data on the server-side for
   things like FastBoot.

Because a route's `model()` hook works by "just returning a promise,"
many new users to Ember start off by returning the result of
`jQuery.ajax()` from this hook.

While this works, jQuery's Promises implementation has historically left
much to be desired. It also doesn't take advantage of the performance
benefits Ember's run loop, and will break in your app's automated tests.

Developers "in the know" use something like `ic-ajax`, which is
bundled with new Ember CLI apps out of the box. The fact that core
functionality—fetching JSON data over HTTP—is not included in the
framework but a third-party library is anathema to Ember's worldview.

Users shouldn't learn about it after they encounter errors in their
tests. Instead, it should be taught as a best practice during the Ember
learning experience.

Because `ic-ajax` is an optional dependency, other tools in the
ecosystem cannot rely on its existence. This makes it tricky for the
broader ecosystem to support things like FastBoot, where jQuery's
`ajax()` isn't available; what do you use to fetch data over HTTP in
Node-land?

# Detailed design

This RFC proposes that we take advantage of the new Services
infrastructure to expose a `network` service that can be used by routes
and other objects (like Ember Data's adapters) that need network access.

## The `network` Service

Objects in the system gain access to the `network` service just like any
other service:

```js
// app/routes/photos.js

import Route from "ember-route";
import service from "ember-service/inject";

export default Route.extend({
  network: service(),

  model() {
    let network = this.get('network');
    return network.fetch('photos.json');
  }
});
```

### `fetch()`

The `network` service exposes a `fetch` method that conforms to the
[WHATWG Fetch specification][fetch]. Under the hood, this method will
use the browser's built-in global `fetch()` function, if available. If
it's not available, a polyfill that uses `XMLHttpRequest` will be used.
In both cases, the service will ensure that the network events and
promise resolution integrate correctly with Ember's run loop.

```js
// app/routes/photos.js

import Route from "ember-route";
import service from "ember-service/inject";

export default Route.extend({
  network: service(),

  model() {
    let network = this.get('network');
    return network.fetch('photos.json')
      .then(response => {
        console.log(response.headers.get('Content-Type'))
        console.log(response.headers.get('Date'))
        console.log(response.status)
        console.log(response.statusText)
      });
  }
});
```

[fetch]: https://fetch.spec.whatwg.org/

By nudging users towards the `fetch()` API, we'll be educating them
about future web standards, a skill they can take to other libraries and
frameworks. By shipping a polyfill with Ember, we'll continue the trend
of letting Ember users take advantage of future web features before
they're widespread in browsers.

## The `ajax` Method

To assist in migrating existing code to use the `network` service, it
will also include an `ajax()` method that is largely compatible with
[jQuery's `jQuery.ajax()` method][ajax].

[ajax]: http://api.jquery.com/jQuery.ajax/

Because jQuery's AJAX API is so expansive, only a subset of the
`jQuery.ajax()` functionality is supported. In deciding what to support,
we should apply an 80/20 rule. The goal is not total API compatibility,
but supporting the majority of common cases.

Under the hood, the `ajax()` method will translate jQuery-style calls to
use its own `fetch()` method. Put another way, `ajax()` is implemented
in terms of `fetch()`. As the ecosystem migrates towards the more modern
`fetch()` API, the `ajax()` method can be deprecated and eventually
removed.

By wrapping `fetch()` in a mostly jQuery-compatible API, we strike a
good balance between allowing users to migrate their code and avoiding
creating a dependency on jQuery.

`ajax()` takes one of two forms: a URL and an options hash, or a single
options hash that contains the URL.

```js
network.ajax('ponies.json', {
  complete() {
    console.log("ponies loaded");
  }
});
```

The `ajax()` method supports a subset of the options supported by jQuery's:

* `accepts`
* `beforeSend`
* `complete`
* `contentType`
* `context`
* `data`
* `dataType` (`"text"` and `"json"` only)
* `error`
* `headers`
* `method`
* `mimeType`
* `password`
* `processData`
* `statusCode`
* `success`
* `timeout`
* `url`
* `username`

The XHR object passed to callbacks such as `success` also provides a
subset of functionality:

* `done()`
* `fail()`
* `always()`
* `then()`
* `readyState`
* `status`
* `statusText`
* `responseText`
* `setRequestHeader()`
* `getAllResponseHeaders()`
* `getResponseHeader()`
* `statusCode()`
* `abort()`

jQuery's prefilters, transports, and converters are not supported.

# Drawbacks

If implemented, this RFC will increase the size and scope of Ember. Some
applications may not need AJAX functionality, instead relying on
something like WebSockets or IndexedDB. In that case, they will pay a
cost for functionality they do not benefit from. This may be mitigated
by tree-shaking.

# Alternatives

## Patch ic-ajax to work in FastBoot

Theoretically, `ic-ajax` could be modified to detect when it is running
in FastBoot and switch to using Node-specific AJAX code in that case.
If this was the recommended course of action for building
FastBoot-compatible addons, it would make `ic-ajax` a de facto
dependency of all applications.

## Ship as an optional addon

Rather than building the `network` service into the framework, it could
be shipped as an optional (but core-supported) addon. The downside of
this is that it's likely to become a de facto dependency of the
ecosystem anyway, and confuse users about what is built-in and what
isn't. In the end, I believe tree-shaking is a better strategy for
reducing file size for unused features than manually breaking up
packages. Manually creating many small packages that users must
carefully curate amounts to error-prone, tedious, manual tree-shaking.

## Do nothing

An option, given how long we've lived with the problem. But given the
importance of FastBoot, I believe it is in the best interest of the
community to allow as many addons as possible to work correctly with it.
AJAX is one of the core things a web app must do, and not having a good
answer for how to do it in FastBoot will likely doom the effort.

# Unresolved questions

## Fidelity to jQuery's API

jQuery's AJAX API is extremely broad. Exposing a compatible API that
follows Ember's Semantic Versioning guarantees is tricky.

On one end of the spectrum, the `network` service's `ajax()` method
could be a small wrapper around jQuery, ensuring that Ember's run loop
is invoked and that the promises returned behave properly.

In FastBoot, we can provide a jQuery-compatible implementation that
works in Node (as we currently do via the [najax package][najax]).

[najax]: https://github.com/najaxjs/najax

The benefits of this approach are that we reduce the amount of work and
documentation that the Ember project must take on. The Ember
documentation can refer to jQuery's, and web developers already familiar
with jQuery will be productive right away.

The downside of this is that it couples us extremely tightly to jQuery,
when we are trying to make an effort to move away from requiring it as a
strict dependency.

It also makes jQuery's implementation a de facto standard. It is likely
that there are small incompatibilities and edge cases between how jQuery
and najax work. Should applications break in FastBoot but work correctly
in the browser, it may be difficult to upstream fixes to resolve the
difference in how edge cases are handled.

On the other end of the spectrum, we can define an entirely new API (or
simply define a subset of the existing jQuery API) that we control. This
would require us to either write our own AJAX code, or do a thorough job
of obfuscating the fact that jQuery is used under the hood.

For example, imagine we design a new API for making a request, but it
returns a promise that resolves to a jQXHR object. Production
applications could come to rely on features that jQuery exposes via
jQXHR objects but that we did not mean to expose as public API. If, in
the future, we replaced the implementation of the `ajax()` method to not
use jQuery, those applications would likely break.

This RFC proposes a middle ground: use a `fetch()` polyfill and write a
small layer that translates jQuery-style `ajax()` calls into `fetch()`
calls. This helps us avoid writing our own network code without risking
exposing jQuery API that users may come to rely on.

## Built-in Services

Should built-in services in Ember be namespaced somehow? As we introduce
new services (`network` this case, things like `routing` in the future),
they may conflict with the names of services in apps or addons.
Upgrading your app to the new version of Ember that contains the service
could cause things to break, violating our Semantic Versioning concerns.

One option here is to detect these conflicts, fall back to the
user-defined service, and emit a warning that they will be unable to use
the new system service until they have renamed the existing user-defined
service.

It is unclear how this would work with addons, however. For example,
imagine you have an `network` service in your app. You upgrade Ember to a
new version that contains the built-in `network` service. You receive a
warning about the conflict but choose to temporarily ignore it. Then,
you install an addon that requires the built-in `network`. If it receives
the user-defined `network` service, the app will fail in surprising and
likely hard-to-diagnose ways.
