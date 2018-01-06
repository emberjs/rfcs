- Start Date: 2018-10-07
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Remove jQuery by default

## Summary

This RFC proposes deprecating those public APIs that are coupled to jQuery, and to finally remove them (with an optional
backport), so Ember apps will be built *by default* without bundling jQuery. 

While [RFC294](https://emberjs.github.io/rfcs/0294-optional-jquery.html), which is already implemented, provides
a way to opt out of jQuery, the intention of this RFC is to push this a step further and essentially move from the
current "included by default, allow opt out" strategy to "excluded by default, allow opt in".

In that way it is not meant as a replacement of the previous RFC, but rather as a continuation and the logical next step.

## Motivation

## Lean by default

This follows the philosophy of making Ember leaner (or *higher octane* if you want), by deprecating unused or 
non-essential APIs.
New apps will be smaller and faster by default, while allowing to opt-in into using jQuery when needed.

### Why the current opt-out strategy is not sufficient

The biggest problem in the current opt-out strategy is that many addons still require jQuery. Many of these usages
seem to be rather "accidental", in that the full power of jQuery is not really needed for the given task, and could be 
rather easily refactored to use only native DOM APIs. But as it is available anyway by default, and it is very convenient, 
authors probably tend to use it without being fully aware of the consequences, that it prohibits jQuery-less builds for 
all its consumers.

In that way the general availability of jQuery *by default* and Ember APIs around it like `this.$()` tend to manifest the 
status quo, the coupling of Ember to jQuery. In fact I could observe an actual *increase* of jQuery usage numbers
(see below), rather than a decrease, which was an intention of the previous RFC. So it is not only a concern of the core 
Ember library to enable jQuery-less builds, but the whole addon ecosystem has to go through that transition.

In that regard early deprecations will help prevent this accidental use of jQuery on the one side, and on the other side
for addons that depend on jQuery already they will provide an incentive and a long enough transition period to refactor 
their jQuery usage to use standard DOM APIs.

### jQuery might still be needed

This RFC does not propose to discourage the use of jQuery. There are legitimate cases where you still want to have it. 
And this is also true for addons, especially those that basically wrap other jQuery-based libraries like jQuery plugins
in an Ember friendly way. For those cases, there should be an *opt-in* path to continue bundling jQuery and to preserve
the existing APIs around it. This is what the `@ember/jquery` package is meant for.

## Transition path

### Add deprecations

All current public APIs that are coupled to jQuery should be deprecated via the usual deprecation process. 
This specifically involves:

* adding a deprecation warning to `Ember.$()`
* adding a deprecation warning to `this.$()` in an `Ember.Component`
* adding a deprecation warning to `this.$()` in component integration tests (based on either the older 
`moduleForComponent()` or the newer `setupRenderingTest()`) 

### Extend `@ember/jquery` package

For apps and addons that have to or choose to still require jQuery, they can add this new package to its dependencies.
This will provide a way to retain the deprecated and later removed APIs. So by adding this to your dependencies this 
would effectively be the way to *opt-in* to require jQuery.

RFC294 already introduced this package, being responsible to include jQuery into the JavaScript bundle. As part of this
RFC the scope of this addon will be extended to also reintroduce the deprecated APIs, but *without* triggering any 
deprecation warnings:
  * `Ember.$()`
  * `this.$()` in a component

As the default `EventDispatcher`, which currently dispatches jQuery events when jQuery is enabled, will eventually 
support native events only (see the Timeline below), the addon also needs to replace it with one that again dispatches
jQuery events for compatibility with existing jQuery-based code. This can happen in a similar way as 
[ember-native-dom-event-dispatcher](https://github.com/rwjblue/ember-native-dom-event-dispatcher) did it, just the other
way around.

**This effectively makes the integration of jQuery a feature of this addon, rather than Ember itself, which is freed from
the burden to care about this.**

So effectively, for the Ember 3.x release cycle, adding this package will not change the behavior in any significant way,
other than removing the mentioned deprecation warnings, as Ember will still have these APIs available. However starting 
with Ember 4.0, which will have these APIs removed and not include jQuery integration features anymore, this 
package will make sure jQuery remains included and it will add the now removed APIs back again, so any jQuery depending
code will continue to work just as before. Also see the timeline below.

As `ember-cli-babel` will currently transform `import $ from 'jquery';` to use `Ember.$` again, it must be made aware of
the `@ember/jquery` package so it tells `babel-plugin-ember-modules-api-polyfill` not to convert those imports to the 
global `Ember.$`. Instead the package itself should provide the necessary shim to make `import $ from 'jquery';` work.

Addons that continue to depend on jQuery would have to list this package as a dependency in their `package.json`, 
to make their consuming app automatically include jQuery and the related APIs in its bundle as mentioned above.
Thereby they make their dependency on jQuery explicit, which in turn helps users to make an educated choice if they 
deem this to be acceptable.

### Timeline

During Ember 3.x:
* migrate the jQuery integration features to the `@ember/jquery` package
* add deprecation warnings as stated above
* update the blueprints for `ember new` to disable jQuery integration by default

Upon Ember 4.0
* remove deprecated functions
* remove the jQuery specific code paths in the `EventDispatcher`

## How we teach this

The Guides already teach native DOM APIs and the new testing APIs, so these do not need to change.

The deprecation messages should link to a deprecation guide, suggesting to preferably use native DOM APIs or install
`@ember/jquery`.

## Drawbacks

### Churn

A vast amount of addons still depend on jQuery, so adding the deprecations will add some considerable churn for the addon
ecosystem. As of writing this, there are:
* [407 addons](https://emberobserver.com/code-search?codeQuery=Ember.%24) using `Ember.$`
* [546 addons](https://emberobserver.com/code-search?codeQuery=this.%24&fileFilter=addon%2Fcomponents) using `this.$` in components
* [994 addons](https://emberobserver.com/code-search?codeQuery=this.%24&fileFilter=tests) using `this.$` in tests

A good amount of that churn can be mitigated by
* existing codemods that migrate tests
* having an easy way, given by the `@ember/jquery` package, to opt-in to continue bundling jQuery, and to restore the 
deprecated APIs, so no further refactorings are required 

## Alternatives

Stick to the current *opt-out* process.

### Deprecation of `this.$()` in old style tests

Rather than specifically deprecating `this.$()` in tests based on the old `moduleForComponent()` based testing APIs, 
the deprecations for the whole suite of old testing APIs could be enforced, as already envisaged in RFC232.
