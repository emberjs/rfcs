- Start Date: 2018-01-11
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Make jQuery optional

## Summary

For the past Ember has been relying and depending on jQuery. This RFC proposes making jQuery optional and having a well 
defined way for users to opt-out of bundling jQuery. 

## Motivation

### Why we don't need jQuery any more

One of the early goals of jQuery was cross-browser normalization, at a time where browser support for web standards was 
incomplete and inconsistent, and Internet Explorer 6 was the dominating browser. It provided a useful and convenient
API for DOM traversal and manipulation as well as event handling, that hid the various browser differences and bugs from
the user. For example `document.querySelector` wasn't a thing at that time, and browsers were using very different event 
models ([DOM Level 0, DOM Level 2 and IE's own proprietary model](https://en.wikipedia.org/wiki/DOM_events#Event_handling_models)). 

But this level of browser normalization is not required anymore, as today's browsers all support the basic DOM APIs well
enough. Even more so that the upcoming Ember 3.0 will drop support for all versions of Internet Explorer except 11.

Furthermore Ember users will need to directly traverse and modify the DOM or manually attach event listeners in very 
special cases only. Most of these low level interactions are taken care of by Ember's templates and its underlying 
Glimmer rendering engine, as well as action helpers or the component's event handler methods.

So having jQuery included by default does not provide that much value to users most of the time, and Ember itself is
expected to be fully functional and tested without jQuery, presumably for the upcoming 3.0 stable release.

### What are the drawbacks of bundling jQuery

The major drawback is the increased bundle size, which amounts to [~29KB](https://mathiasbynens.be/demo/jquery-size) 
(minified and gzipped). This not only increases the loading time, but also parse and compile times, thus increasing the 
total time to interactive. This is especially true for mobile devices, where slow connectivity and weak CPU performance 
is not uncommon.

Having jQuery not included will improve the suitability of Ember for mobile applications considerably. Even
if the raw number is not that huge, it all adds up. And it plays together with other efforts to make leaner Ember builds
possible, like enabling tree shaking with the new [Module API](https://github.com/emberjs/rfcs/blob/master/text/0176-javascript-module-api.md),
moving code from core to addons (e.g. the [`Ember.String` deprecation](https://github.com/emberjs/rfcs/blob/master/text/0236-deprecation-ember-string.md))
or the ["Explode RFC"](https://github.com/emberjs/rfcs/blob/explode/text/0000-explode.md). In that regard removing the 
dependency on jQuery is a rather low hanging fruit with an high impact.

### But this is already possible, why this RFC?

There is indeed a somewhat quirky way to build an [app without jQuery](https://github.com/rwjblue/no-jquery-app) even
today.
Although this *happens* to work, it is not sufficient to consider this officially supported for these reasons:
* Ember itself must be fully tested to work without jQuery
* the public APIs that depend on and/or expose jQuery need to have some well defined behavior when jQuery is not 
available
* there should be a way to technically opt-out (other than fiddling with [`vendorFiles`](https://github.com/rwjblue/no-jquery-app/commit/34c40fc2cfc5e2ce0c39e5e906448c46af699d26))
that is easier to use, understand and maintain
* addons should mostly default to not use jQuery, to make removing jQuery practically possible for their consuming
apps

## Detailed design

### Remove internal jQuery usage

As of writing this, there are [major efforts](https://github.com/emberjs/ember.js/issues/16058) underway to remove and 
cleanup the Ember codebase and especially its tests from jQuery usage. Having a way to fully test Ember without jQuery 
is a prerequisite to officially support jQuery being optional. When this is done, it will enable a "no jQuery" mode, 
that will make it not use jQuery anymore, but only native DOM APIs.

### Add an opt-out flag

There should be a global flag that will toggle the optional jQuery integration (true by default). When this is disabled,
it will make Ember CLI's build process *not* include jQuery into the `vendor.js` bundle, *and* it will explicitly put 
Ember itself into its "no jQuery" mode.

The flag itself will not be made a public API. Rather it will be handled by a privileged addon, that will allow to 
disable the integration flag, thus to opt out from jQuery integration. This approach is in line with 
[RFC 278](https://github.com/emberjs/rfcs/pull/278) and [RFC 280](https://github.com/emberjs/rfcs/pull/280), to allow 
for some better implementation flexibility.

### Introduce `@ember/jquery` package

Currently Ember CLI itself is importing jQuery into the app's `vendor.js` file. To decouple it from this task, and
to allow for some better flexibility in the future, the responsibility for importing jQuery is moved to a dedicated 
`@ember/jquery` addon.

To not create any breaking changes, Ember CLI will have to check the app's dependencies for the presence of this addon. 
If it is not present, it will continue importing jQuery *unless* the jQuery integration flag is disabled.
If it is present, it will stop importing jQuery at all, and delegate this responsibility to the addon.

To nudge users to install `@ember/jquery` when they need jQuery, some warning/deprecation messages should be issued when
the addon is *not* installed and the integration flag is either not specified or is set to true. To ease 
migration the addon should be placed in the default blueprint (until an eventual more aggressive deprecation of 
jQuery). Only in the case the app is actively opting out of jQuery integration the addon is not needed.

The addon itself has to make sure the Ember CLI version in use is at least the one that introduced the above mentioned
logic, to prevent importing jQuery twice.

### Assertions for jQuery based APIs

Apart from testing (see below), Ember features some APIs that directly expose jQuery, which naturally cannot continue 
to work without it. For these APIs some assertions have to be added when running in "no jQuery" mode (and not in 
production), that provide some useful error messages for the developer:

* `Ember.$()`
  should throw an assertion stating that jQuery is not available.
* `this.$()` in components
  should throw an assertion stating that jQuery is not available and that `this.element` and native DOM APIs should be
  used instead.

### Testing

Ember's test harness has been based on jQuery for a long time. Most global acceptance test helpers like `find` or 
`click` rely on jQuery. For integration tests the direct use of jQuery like `this.$('button').click()` to trigger 
events or assert the state of the DOM is still the standard, based on `this.$()` returning a jQuery object representing
the rendered result of the tests `render` call.

To be able to reliably run tests in a jQuery-less world, we need to run our tests without jQuery being included,
so our test harness has to work without jQuery as well.

Fortunately this is well underway already. [ember-native-dom-helpers](https://github.com/cibernox/ember-native-dom-helpers)
introduced native DOM test helpers for integration and acceptance tests as an user space addon. The recent acceptance 
testing [RFC 268](https://github.com/emberjs/rfcs/blob/master/text/0268-acceptance-testing-refactor.md) provides 
similar test helpers, implemented in the `@ember/test-helpers` package, and envisages deprecating the global test 
helpers.

However while the existing jQuery based APIs are still available, when these are used without jQuery they have to throw
an assertion with some meaningful error message:

* global acceptance test helpers that expect jQuery selectors (which are a potentially incompatible superset of standard
CSS selectors)

* `this.$()` in component tests, provided currently by `@ember/test-helpers` in `moduleForComponent` and 
`setupRenderingTest`

In both cases the error message should state that jQuery is not available and that the native DOM based test helpers
of the `@ember/test-helpers` package should be used instead.

The transitioning to these new test helpers can be eased through a codemod. For `ember-native-dom-helpers` there already
exists  [ember-native-dom-helpers-codemod](https://github.com/simonihmig/ember-native-dom-helpers-codemod), which 
could be adapted to the very similar RFC 268 based interaction helpers in `@ember/test-helpers`.

### Implementation outline

The following outlines how a possible implementation of the jQuery integration flag *could* look like. This
is just to provide some additional context, but is *intentionally not* meant to be normative, to allow some flexibility 
for the actual implementation.

The addon that will handle the flag is expected to be [ember-optional-features](https://github.com/emberjs/ember-optional-features),
which will read from and write to a `config/optional-features.{js,json}` file. This will hold the `jquery-integration` 
flag (amongst others). This flag in turn will be added to the `EmberENV` hash, which will make Ember go into its 
"no jQuery" mode when set to `false`. 

Ember CLI and the `@ember/jquery` addon will also look for `jquery-integration` in this configuration file, and will 
opt-out of importing jQuery when this file is present and the flag is set to `false`.

## How we teach this

### Guides

The existing "Managing Dependencies" chapters in the Ember Guides as well as on ember-cli.com provide a good place to 
explain users how to set the jQuery integration flag by means of the mentioned [privileged addon](#add-an-opt-out-flag) 
that handles this flag. 

The section on components should be updated to remove any eventually remaining references to `this.$`, to not let users 
fall into the trap of creating an implicit dependency on jQuery by "accidental" use of it. These should be changed to 
refer to their native DOM counterparts like `this.element` or `this.element.querySelector()`.

The section on acceptance tests will have been updated as per [RFC 268](https://github.com/emberjs/rfcs/blob/master/text/0268-acceptance-testing-refactor.md)
to use the new `@ember/test-helpers` based test helpers instead of the jQuery based global helpers.

The section on component tests should not use `this.$()` anymore as well, and instead also according to [RFC 268](https://github.com/emberjs/rfcs/blob/master/text/0268-acceptance-testing-refactor.md) 
use `this.element` to refer to the component's root element, and use the new DOM interaction helpers instead of jQuery 
events triggered through `this.$()`. 

### Addon migration

One of the biggest problems to easily opt-out of jQuery is that many addons still depend on it. Many of these usages 
seem to be rather "accidental", in that the full power of jQuery is not really needed for the given task, and could 
be fairly easily refactored to use only native DOM APIs.

For this reason this RFC encourages addon authors to not use jQuery anymore and to refactor existing usages whenever 
possible! This certainly does not apply categorically to all addons, e.g. those that wrap jQuery plugins as 
components and as such cannot drop this dependency.

#### ember-try

`ember-try`, which is used to test addons in different scenarios with different dependencies, should provide some means
to define scenarios without jQuery, based on the jQuery integration flag introduced in this RFC.

Furthermore the Ember CLI blueprint for addons should be extended to include no-jQuery scenarios by default, to make
sure addons don't cause errors when jQuery is not present.

#### emberobserver.com

It would be very helpful to have a clear indication on [emberobserver.com](https://emberobserver.com/) which 
addons depend on jQuery and which not. This would benefit users as to know which addons they can use without 
jQuery, but also serve as an incentive for authors to make their addons work without it.

Given the jQuery integration flag introduced in this RFC, this paves the way to automatically detect addons that are 
basically declaring their independence from jQuery by having this flag set to `false`  (in their own repository).

## Drawbacks

### Churn

A vast amount of addons still depend on jQuery. While as far as this RFC is concerned no jQuery based APIs will be
deprecated and the default will still be to include jQuery, addons are nevertheless encouraged to remove their 
dependency on jQuery, which will add some considerable churn to the addon ecosystem. As of writing this, there are:
* [475 addons](https://emberobserver.com/code-search?codeQuery=Ember.%24) using `Ember.$`
* [479 addons](https://emberobserver.com/code-search?codeQuery=this.%24&fileFilter=addon%2Fcomponents) using `this.$` in
components
* [994 addons](https://emberobserver.com/code-search?codeQuery=this.%24&fileFilter=tests) using `this.$` in tests

Among these are still some very essential addons like `ember-data`, which still relies on `$.ajax`, see 
[#5320](https://github.com/emberjs/data/issues/5320).

A good amount of that churn can be mitigated by having a codemod that migrates tests (see "Testing" above).

## Alternatives

Continue to depend on jQuery.

## Unresolved questions

### jquery.Event vs. native events

Event handler methods in components will usually receive an instance of `jquery.Event` as an argument, which is very
similar to native event objects, but not exactly the same. To name a few differences, not all properties of the native 
event are mapped to the jQuery event, on the other hand a jquery event has a `originalEvent` property referencing the
native event. 

The updated event dispatcher in Ember 3.0 is capable of working without jQuery (similar to what 
`ember-native-dom-event-dispatcher` provided for Ember 2.x). When jQuery is not available, it will naturally not be 
able to pass a `jquery.Event` instance but a native event instead. This creates some ambiguity for addons, as they 
cannot know in advance how the consuming app is built (with or without jQuery).

* should addon authors be encouraged to add checks for the type of event (like `event instanceof jQuery.Event`) if 
they rely on jQuery event specifics? (which is already the case when they just need access to the native event 
`originalEvent`)
* should we aim to remove that ambiguity and pass only native events at some time, regardless of jQuery being available
or not? This would be a breaking change, so can be done for Ember 4.0 only. If not, the type checks mentioned above 
would have to stay around basically forever
* for the above case, one *could* add deprecation warnings when jQuery specific properties like `originalEvent` are 
accessed. However this can be misleading, as it may be required to use these jQuery specifics if a jQuery event is 
passed, so there is technically no other way to prevent the deprecation warning to be triggered. 
* is there a way to start passing *only* native events, which are augmented by jQuery specific properties like
`originalEvent` (e.g. `event.originalEvent = event`) to not break things, thus following SemVer while still 
transitioning to native events only. This would allow all addons to assume native events without any type checks. 
Something similar seems to have been discussed for the 
[jQuery 4.0 Event Design](https://github.com/jquery/jquery/wiki/jQuery-4.0-Event-Design#avoid-the-need-for-a-jqueryevent-wrapper),
but seems to be controversial and possibly error prone.
