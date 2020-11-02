---
Start Date: 2015-05-24
RFC PR: https://github.com/emberjs/rfcs/pull/58
Ember Issue: https://github.com/emberjs/ember.js/pull/11393

---

# Summary

This RFC outlines a new strategy for the registration of helpers in Ember 1.13.
In previous versions of Ember, helper lookup was a two-phase process of:

* Look in a whitelist of registered helpers. If in the whitelist, resolve that
  path in the container.
* If the path has a dash, try to resolve it in the container
* If the container does not have a factory for this path, treat the path as a
  bound value.

This logic runs for every `{{somePath}}` in an Ember application.

This proposal attempts to simplify and unify that logic in a a single pass
against a whitelist, thus removing the special behavior of dashed paths.
Additionally, it attempts to design a solution that removes the current
`registerHelper` ceremony for undashed helpers.

# Motivation

In [RFC #53](https://github.com/emberjs/rfcs/pull/53) a new API for helpers is
outlined. This RFC presumes helpers will continue to have the naming
requirement of including a dash character.

The dash requirement for helpers exists for two reasons:

* For every `{{path}}` in an Ember application, it must be decided if that path
  is a bound value, component, or helper. Component and helper lookup (the
  discovery of a class or template) is lazy in Ember, thus for every `{{path}}`
  a lookup for that string in the container is required. Container lookups
  (the first time) are fairly slow, and performing this lookup for every
  path may significantly impact initial render time. Thus, helpers are either
  added to a whitelist (with `registerHelper`) or require a way to differentiate
  themselves from the majority of data-binding cases (the dash).
* In Ember 1.x, components were treated as helpers for certain code paths. This
  made the dash requirement for components a natural extension to helpers.

The Glimmer engine has removed some of these concerns and limitations.

Addon authors and app authors have both felt a need for non-dashed helper
names, for example `{{t 'some-string-to-translate'}}`. New developers to Ember
often find the dash requirement arbitrary and the `registerHelper` work around
difficult to understand and use.

For the new helper API to provide feature parity with APIs available to addon
authors in 1.12, a path to dashless helpers must be present in 1.13.

Given that a solution exists that addresses the performance concern, dropping
the dash requirement would resolve a significant amount of developer pain and
confusion.

# Detailed design

At application boot, all known helper items (according to the resolver) are
iterated and added to a `helper-listing` service. This service is merely a
Set object with the names of all helpers.

When handling a `{{path}}`, the `helper-listing` service is consulted for the
presence of that `path`. If it is present, the path is looked up
on the container as a helper and the helper is used. Dashed paths are treated
no differently than any other path (for helpers).

### Boot time discovery

To discover what paths may be helpers in Ember-CLI, the module names are
iterated. For example:

```
not helper: app/components/foo-bar/component
not helper: app/controllers/foo-bar
not helper: app/foo-bar/route
helper "t": app/t/helper
helper "t": app/helpers/t
helper "foo-bar": app/helpers/foo-bar
helper "foo/bar": app/helpers/foo/bar
```

In a globals-mode application, The app namespace is iterated:

```
not helper: App.FooBarComponent
not helper: App.FooBarController
not helper: App.FooBarRoute
helper "t": App.THelper
helper "foo-bar": App.FooBarHelper <- should dasherize
```

In both cases **the resolver is responsible for providing a list of modules
by type**. The proposed API is `eachOfType`, here with Ember-CLI as an example:

```js
// Given helperListing as a Set:
resolver.eachOfType(‘helper’, function(parsedName, item) {
  helperListing.add(parsedName.fullName);
})
```

In Ember-CLI, the `app/` tree of an addon is merged with the app tree of an
application. This means for a helper like `t` to be discovered, nothing besides
adding it to `app/helpers/t.js` must be done.

In 1.13, this will impact existing apps by discovering all helpers regardless
of if `registerHelper` has been called. This is a small behavior change that
should match intent, and will not impact sanely written apps.

Note that only the path of the helper is added to the listing. During discovery,
the helper is not looked up from the container, instead lookup still occurs
at render time.

The helper listing is intended to be a private service in Ember, and will be
registered at `services:-helper-listing`. If the discovery semantics described
here are not sufficient for some edge-cases, wrapping this service in a
public API on application instances may be required.

### Render-time lookup and use

Let us consider how a path is rendered. For example:

```hbs
{{date}}
```

* The `service:-helper-listing` service is fetched
* The path `date` is checked for on the listing: `helperListing.has(path)`
* If the path is not in the listing, `date` is treated like a bound value
* If the path is in the listing, the helper is looked up from Ember's
  container as `helper:date`
* depending on the instance returned from the factory (a helper, shorthand
  helper, or legacy `Ember.Handlebars` or `Ember.HTMLBars._registerHelper`
  helper) the proper invocation for that helper is executed

Every rendered path will hit the `helper-listing` service, but the check
against a well-implemented Set should be inexpensive.

# Drawbacks

Removing the dash requirement will likely result in a larger number of naming
conflicts between addons and apps than has existed before now. In general,
encouraging verbose helper names may mitigate this concern. Long term, there
have been several discussions to date about how to implement namespaces in
Ember templates and for Ember engines.

That the helper listing is eagerly discovered at application boot time may
impact the design of Ember engines and lazy-loading parts of an app. The
discovery cache may need to be flushed and re-generated, however this limitation
already exists for the container lookup itself (which caches failures).

That the helper listing is not based on the container means helpers registered,
but not added to the listing because of non-standard naming, may need to
manually register against the private helper listing API.

# Alternatives

Instead of a new across-the-board solution, Ember could continue to use a
`registerHelper` pattern very similar to what exists today. This would
perpetuate the existing pain, but would perhaps be more similar to what devs
already know.

# Unresolved questions

The exact timing of helper discovery in Ember-CLI and globals mode has not been
decided.
