---
Start Date: 2015-06-30
RFC PR: https://github.com/emberjs/rfcs/pull/65
Ember Issue: (leave this empty)

---

# Summary

Deprecations and warnings in Ember.js should have configurable runtime handlers.
This allows default behavior (logging, raise when `RAISE_ON_DEPRECATION` is true)
to be overridden by an enviornment (Ember's tests), addon, or other tool
(like the Ember Inspector).

Ember-Data and the Ember Inspector have both requested a public
API for changing how deprecation and warning messages are handled. The requirements
for these and other requests are complex enough that deferring the message
behavior into a runtime hook is the suggested path.

# Motivation

`Ember.deprecate` and `Ember.warn` usually log messages. With `ENV.RAISE_ON_DEPRECATION`
all deprecations will throw an exception. In some scenarios, this
is less than ideal:

* Ember itself needs a way to silence some deprecations before their usage
  is completely removed from tests. For example, many view APIs in Ember 1.13.
* The Ember inspector desires to raise on specific deprecations, or silence
  specific deprecations.
* Ember-Data also desires to silence some deprecations in tests

In [PR #1141](https://github.com/emberjs/ember.js/pull/11419)
a private log level API has been introduced, which allows finer grained control
if specific deprecations should be logged, throwing an error or be silenced
completely. For example:

```js
Ember.Debug._addDeprecationLevel('my-feature', Ember.Debug._deprecationLevels.LOG);
// ...
Ember.deprecate("x is deprecated, use Y instead", false, { id: 'my-feature' });
```

Initially a public version of this API was discussed, but it quickly became
clear that a runtime hook provided more flexibility without incurring the
cost of a complex log-level API.

Note that "runtime" refers to Ember itself. A custom handler could be injected
into Ember-CLI's template compilation code. "runtime" in this context still
refers to handling deprecations raised during compilation.

# Detailed design

A handler for deprecations can be registered. This handler will be called
with relevent information about a deprecation, including guarantees about
the presence of these items:

* The deprecation message
* The version number where this deprecation (and feature) will be removed
* The "id" of this deprecation, a stable identifier independent of the message

Additionally, an application instance may be passed with the options. An example
handler would look like:

```js
import { registerHandler } from "ember-debug/deprecations";

registerHandler(function deprecationHandler(message, options) {
  // * message is the deprecation message
  // * options.until is the version this deprecation will be removed at
  // * options.id is the canonical id for this deprecation
  if (options.until === "2.4.0") {
    throw new Error(message);
  } else {
    console.log(message);
  }
});
```

Warnings are similar, but will not recieve an `until` value:

```js
import { registerHandler } from "ember-debug/warnings";

registerHandler(function warningHandler(message, options) {
  // * message is the warning message
  // * options.id is the canonical id for this warning
  if (options.id !== 'view.rerender-on-set') {
    console.log(message);
  }
});
```

##### chained handlers

Since several handlers may be registered, a method of deferring to a previously
registered handler must be allowed. A third option is passed to handlers, the
function `next` which represents the previously registered handler.

For example:

```js
import { registerHandler } from "ember-debug/deprecations";

registerHandler(function firstDeprecationHandler(message, options, next) {
  console.warn(message);
});

registerHandler(function secondDeprecationHandler(message, options, next) {
  if (options.until === "2.4.0") {
    throw new Error(message);
  }
  next(...arguments);
});
```

The first registed handler will receive Ember's default behavior as `next`.

##### new assertions for deprecate and warn

Ember's APIs for deprecation and warning do not currently require any information
beyond a message. It is proposed that deprecations be **required** to pass
the following information:

* Message
* Test
* Canonical id (with a format of `package-name.some-id`)
* Release when this deprecation will be stripped

For example:

```
import Ember from "ember";

Ember.deprecate("Some message", false, {
  id: 'ember-routing.query-params',
  until: '3.0.0'
});
```

If this information is not present and assertion will be made.

Warnings likewise will be required to pass a canonical id:

```
import Ember from "ember";

Ember.warn("Some warning", {id: 'ember-debug.something'});
```

##### default handlers

The default handler for deprecation should be quite simple, and mirrors current
behavior:

```js
function defaultDeprecationHandler(message, options) {
  if (Ember.ENV.RAISE_ON_DEPRECATION) {
   throw new Error(format(message, options));
  } else {
   console.log(format(message, options));
  }
}
```

The default handler for warnings would be simple `console.log`.

# Drawbacks

By not providing a robust log-level API, we are punting complexity to the
consumer of this API. For a low-level tooling API such as this one, it seems
and appropriate tradeoff.

# Alternatives

Each app can stub out `deprecate` and `warn`.

# Unresolved questions

`RAISE_ON_DEPRECATION` could be considered deprecated with this new API.
