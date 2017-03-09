- Start Date: 2016-06-14
- RFC PR: [#55](https://github.com/ember-cli/rfcs/pull/55)

# Summary

This RFC proposes extending the `app.import` API to consume anonymous AMD modules. It accompanies [ember-cli PR 5976](https://github.com/ember-cli/ember-cli/pull/5976).

# Motivation

AMD modules come in two flavors: named and anonymous. _Anonymous AMD_ is the more portable distribution format, but it typically requires preprocessing into _named AMD_ before it can be included into an application.

    /* Anonymous AMD Examples */

    // direct value
    define({ color: 'black' });

    // function returning value
    define(function() { return { color: 'black' }; });

    // function returning value, with declared dependencies
    define(["jquery", "moment"], function(jQuery, moment) {
      return {
        injectTime: function() {
          jQuery('#time-box').html(moment().format('HH:MM'));
        }
      }
    });

    /* Named AMD Examples */

    // direct value
    define('my-config', { color: 'black' });

    // function returning value
    define('my-config', function() { return { color: 'black' }; });

    // function returning value, with declared dependencies
    define('time-utils', ["jquery", "moment"], function(jQuery, moment) {
      return {
        injectTime: function() {
          jQuery('#time-box').html(moment().format('HH:MM'));
        }
      }
    });


Today, ember-cli users can add arbitrary third-party _named AMD_ modules into their application via:

    app.import('/path/to/module.js');

But this does not support _anonymous AMD_ modules, which is annoying because _anonymous AMD_ is the better format for library distribution and is widely used.

# Detailed design

In order to de-anonymize AMD, it's necessary to choose a name for the module for use within a given application. So I propose extending the:

    app.import('/path/to/module.js');

API with an additional argument:

    app.import('/path/to/module.js', {
      using: [
        { transformation: 'amd', as: 'some-dep' }
      ]
    });

`using` provides a list of transformations. Each transformation is identified by its `transformation` property. Any other properties are treated as arguments to the transformation implementation -- they are opaque to ember-cli. Transformations will run in the given order.

In this particular case, the `amd` transformation will run and receive the argument `{as: 'some-dep'}`.

The exactly meaning of the `amd` transformation is: within this Javascript file, any call(s) to the global `define()` function will be intercepted and the given module name (`some-dep` in the above example) will be prepended to the argument list.

[A complete implementation is available here](https://github.com/ember-cli/ember-cli/pull/5976). (As of this edit it lags behind updates to this RFC.)

# Learning

An appropriate place to document this feature is [here](https://ember-cli.com/user-guide/#standard-amd-asset). That existing documentation is silent on the distinction between named and anonymous AMD, which probably trips people up.

# Drawbacks

I am not attempting to specify static error detection, mostly because doing that well would require fully parsing and understanding the imported module, which is likely to be more expensive and fragile.

Examples of static errors that would theoretically be nice to detect would be the presence of a _named AMD_ module in the file, the lack of any AMD module in the file, or the present of multiple _anonymous AMD_ modules in the file.

The current implementation causes any sourcemap information inside the imported file to be discarded (you don't get an invalid sourcemap, but you lose detail).

I have not specified a pluggable way to add additional transformations. My intent is to reserve space in our public API so that future extraction and pluggability is fully backward compatible.

# Alternatives

Many libraries fall back to global variables if they cannot detect a valid AMD loader. I suspect this is the most common alternate pattern that's in use in the community.

Some applications include their own manually written shims in `vendor` or elsewhere.

# Unresolved questions

We should confirm that my implementation performs well in apps with very large dependency directories.
