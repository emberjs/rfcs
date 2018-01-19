- 2018-01-17
- RFC PR: 0297
- Ember Issue: (leave this empty)

# Deprecation of Ember.Logger

## Summary

This RFC recommends the deprecation and eventual removal of `Ember.Logger`.

## Motivation

There are a variety of features of Ember designed to support old browsers,
features that are no longer needed. `Ember.Logger` came into being because
the browser support for the console was inconsistent. In some browsers,
like Internet Explorer 9, the console only existed when the developer tools
panel was open, which caused null references and program crashes when run
with the console closed. `Ember.Logger` provided methods that would route to 
the console when it was available.

With Ember 3.x, Ember no longer supports these older browsers, and hence this
feature no longer serves a purpose. Removing it will make Ember smaller and 
lighter.

## Detailed design

For the most part, this is a 1:1 substitution of the global `console` object 
for `Ember.Logger`. However, Node only added support for `console.debug` in 
Node version 9. To support earlier versions of Node, our codemod will need to 
use `console.log`, rather than `console.debug`, as the replacement for 
`Logger.debug`, except for add-ons that specify that they only support Node 
version 9 and beyond.

When a project is built for production, `Ember.Logger` calls become a no-op. 
Replacing `Ember.Logger` calls with console calls will require some other 
mechanism to suppress logging in production builds. Fortunately, the 
`babel-plugin-transform-remove-console` will suppress console 
interaction, a far more comprehensive solution than relying upon
`Ember.logger`. We will need to supply instructions for setting this up (for
production builds only) in the `ember-cli` Broccoli pipeline.


### Within the framework

Remove the following direct uses of `Ember.Logger` from the ember.js and 
ember-data projects: 

* `ember-debug`:
    *  deprecate (`ember-debug\lib\deprecate.js`) - `Logger.warn`
    *  debug (`ember-debug\lib\index.js`) - `Logger.info`
    *  warn (`ember-debug\lib\warn.js`) - `Logger.warn`
* `ember-routing` (`ember-routing\lib\system\router.js`):
    *  transitioned to - `Logger.log`
    *  preparing to transition to - `Logger.log`
    *  intermediate-transitioned to - `Logger.log`
* `ember-testing`:
    *  Testing paused (`ember-testing\lib\helpers\pause_test.js`) - `Logger.info`
    *  Catch-all handler (`ember-testing\lib\test\adapter.js`) - `Logger.error`
* `ember-data`:
    *  `tests\test-helper.js`- `Logger.log`

Adjust all test code that redirects logging and sets it back:

* `ember\tests\routing\basic_test.js` (adjust)
* `ember-application\tests\system\dependency_injection\default_resolver_test.js` (adjust)
* `ember-application\tests\system\logging_test.js` (remove?)
* `ember-glimmer\tests\integration\helpers\log-test.js` (remove?)

Note: None of the uses of `Ember.Logger` in `ember.js` or `ember-data` involve
`Ember.debug`, so that issue doesn't affect the Ember.js code directly.

Add deprecation warnings to the implementation: `ember-console\lib\index.js`.
Bear in mind that the deprecation mechanism currently calls `Logger.warn`, so 
that code should be changed _first_ or this change will be very difficult 
to debug.

### Codemod 

Provide a codemod that developers can use to switch references to the methods 
of `Ember.Logger` to use the corresponding `console` methods instead. I 
will definitely need help here.

### Add-On Developers

The following high-impact add-ons (9 or 10 or a * on EmberObserver) use 
`Ember.Logger` and should probably be given an early heads-up to adjust 
their code to use `console` before this RFC is implemented. This will limit 
the level of pain that their users experience when the deprecation is released.

In the order of their number of references to `Ember.Logger`:

* `ember-concurrency` (15)
* `ember-cli-deprecation-workflow` (9)
* `ember-stripe-service` (9)
* `semantic-ui-ember` (7)
* `ember-resolver` (6)
* `ember-cli-page-object` (4) 
* `ember-cli-sentry` (3)
* `ember-islands` (3)
* `ember-states` (3)
* `ember-cli-pagination` (2)
* `ember-cli-clipboard` (1)
* `ember-cli-fastboot` (1)
* `ember-elsewhere` (1)
* `ember-i18n` (1)
* `ember-simple-auth-token` (1)
* `ember-svg-jar` (1)
* `liquid-fire` (1)

For details, see https://emberobserver.com/code-search?codeQuery=Ember.Logger.

## How we teach this

### Communication of change

We need to inform users that `Ember.Logger` will be deprecated and in what 
release it will occur. 

### Official code bases and documentation

We do not currently actively teach the use of `Ember.Logger`. We will need to 
remove any passing references to `Ember.Logger` from the Ember guides 
from the Super Rentals tutorial, and anywhere else it appears on the website.

Once it is gone from the code, we also need to verify it no longer appears in 
the API listings. 

We must supply instructions for using `babel-plugin-transform-remove-console`
in `ember-cli` so it is only applied in production builds.

We must provide an entry in the deprecation guide for this change:
* offering instruction for using the codemod to perform the change automatically
with before and after code samples.
* documenting the use of `babel-plugin-transform-remove-console` in the 
`ember-cli` build to suppress console calls in production.
the console calls in production code. 
* describing the issue with using console.debug on node versions 
earlier than Node 9 and what provision the codemod has made to deal with it.
* describing alternative ways of dealing with eslint's `no-console` messages.

## Drawbacks

191 add-ons in Ember Inspector are using `Ember.Logger`. It has been there and 
documented for a long time. So this deprecation will cause some level of change 
on many projects. 

This, of course, can be said for almost any deprecation, and Ember's 
disciplined approach to deprecation has been repeatedly shown to ease things. 
Providing a codemod to replace `Ember.Logger` calls with the corresponding 
console calls should make this transition relatively painless. Also, only 
twenty of those add-ons have more than six references to `Ember.Logger`. 
If this is characteristic of the user base, the level of effort to make 
the change, even by hand, should be very small for most users.

Those using `Logger.debug` as something different from `Logger.log`may have at 
least a theoretical concern. Under the covers `Logger.debug` only calls 
`console.debug` if it exists, calling `console.log` otherwise. The only 
platform where the difference between the two is visible in the console is on 
Safari. We can encourage folks with a tangible, practical concern about this to
speak up during the comment period, but I don't anticipate this will have much 
impact.

## Alternatives

1. Leave things as they are, perhaps providing an `@ember/console` module 
interface.

2. Extract `Ember.Logger` into its own (tiny) `@ember/console` package as 
a shim for users.

## Unresolved questions

How do we deal with `Logger.debug` in the codemod? Do we provide separate 
options for those who might use Node versions earlier than 9 and those who 
are confident they will only use Node version 9 or later? Do we have the 
codemod inject a polyfill for console.debug that calls console.log? Do we
provide one separately for the user to apply? Or does that become a fastboot 
concern, since that's the primary driver for running ember projects in node.

What do we do about the eslint `no-console` flag? Some developers are 
using `Ember.Logger` right now to work around it. Others are providing 
their own logging service. If we encourage the use of a Babel plugin 
to strip console calls, should we remove `no-console` from the default 
flags that `ember-cli` ships?
