---
Start Date: 2017-12-11
RFC PR: https://github.com/emberjs/rfcs/pull/278
Ember Issue: https://github.com/emberjs/ember.js/pull/15974

---

# Summary

Introduce a low-level "flag" to remove the automatic wrapper `<div>` for
template-only components (templates in the `components` folder that do not
have a corresponding `.js` file).

In other words, given there is NO `app/components/hello-world.js` and there
exists `app/templates/components/hello-world.hbs` which contains the
following markup:

```hbs
Hello world!
```

When this template-only component is invoked as `{{hello-world}}` with the
flag unset or disabled (i.e. today's semantics), Ember will render:

```html
<div id="ember123" class="ember-view">Hello world!</div>
```

When the flag is enabled, the same invocation will render:

```html
Hello world!
```

# Motivation

With today's component system (i.e. `Ember.Component`), a wrapper element (a
`div` by default, along with an ID like `ember123` and the `ember-view` class)
is automatically added for every component.

Customizing this wrapper element (such as changing the tag name – or removing
it altogether) requires making changes to the component's JavaScript class,
such as:

```js
import Component from "@ember/component";

export Component.extend({
  tagName: "footer",
  classNames: ["legalese"]
});
```

While we acknowledge this API is quite cumbersome, it is sufficient to "get
things done" for regular components, and Glimmer Components will address
the usability aspect once they land.

However, this API does not work for template-only components, as they do
not have a component JavaScript class by definition. Therefore, in practice,
template-only components always come with a `<div>` wrapper, along with the
default `id` and `class` attributes, with no obvious ways to customize it.

This is quite problematic, as it is often desirable to use a template-only
component to organize content that requires a certain markup structure. The
most common workaround for this problem is to use a partial instead, which
comes with [a host of issues](https://github.com/emberjs/rfcs/pull/262). I
will discuss other workarounds in the section below.

This RFC proposes to add a global flag to remove this wrapper element around
template-only components. This will allow the component author to specify the
wrapper element in the component template, offering direct control over the
tag name and other attributes. It would also allow the component to have more
than one top-level element, or none at all.

In other words, this flag changes template-only components in the app to have
"Outer HTML" semantics. _What you type is what you get._

Notably, [Glimmer Components](https://glimmerjs.com/guides/templates-and-helpers)
have adopted the "Outer HTML" semantics long ago and the experience has been
very positive. This would be one of the first pieces of the Glimmer.js experiment
to make its way into Ember. We think this feature is small, self-contained but
useful enough to be integrated back into Ember at this point.

If accepted, this RFC will fully subsume the [Non-context-shifting partials](https://github.com/emberjs/rfcs/pull/262)
RFC. We can therefore (at a later time, in a separate RFC) explore deprecating
partials in favor of wrapper-free template-only components.

# Detailed design

## API Surface

We should not expose the flag directly as a public API. Instead, we should
abstract the flag with a "privileged addon" whose only purpose is to enable
the flag. Applications will enable the flag by installing this addon. This
will allow for more flexibility in changing the flag's implementation (the
location, naming, value, or even the existence of it) in the future. From the
user's perspective, it is the _addon_ that provides this functionality. The
flag is simply an internal implementation detail.

We have done this before in other cases (such as the legacy view addon during
the 2.0 transition period), and it has generally worked well.

When landing this feature, it will be entirely opt-in for existing apps, but
the Ember CLI application blueprint should be updated to include the addon by
default. At a later time, we should provide another addon that _disables_ the
flag explicitly (installing both addons would be an install-time error). At
that time, we will issue a deprecation warning if the flag is *not set*, with
a message that directs the user to install one of the two addons.

## Single Global "Flag"

The proposed flag will be truly global in scope. That is, setting this flag
will change the semantics of all template-only components in the entire app,
even for components that were included by addons.

However, we believe this would not affect any addon components in practice,
as the predominant pattern for addons to expose components currently
necessitates a JavaScript class. Addon authors would create the component
(with or without a JavaScript class) in the `/addon` folder, but exposing
it for consumption in apps requires creating a corresponding JavaScript class
in the `/app` folder to "re-export" the component. Therefore, in practice,
it is not actually possible for addons to have a truly template-only
component today (something to address in a future RFC).

## Leakage Of `Ember.Component` Semantics

While the primary purpose of this flag is to remove the wrapper element from
template-only components, there are a few other observable semantics changes
that comes with it as well.

Currently, template-only components are "backed" by an instance of `Ember.Component`.
That is, Ember will create an instance of `Ember.Component` and set it as the
`{{this}}` context for the template.

With the flag enabled, there will be *no* component instance for the template
and `{{this}}` will be set to `undefined` (or `null`, perhaps). This would
improve performance for template-only components significantly.

Since there is no JavaScript file for the component, this is only observable
in a few limited ways:

1. The most noticable artifact is the component's arguments will not be
   auto-reflected on the component instance (as there is no component
   instance at all). Therefore, the only way to access the component's
   arguments is to use the `{{@foo}}` syntax proposed in [RFC #276](https://github.com/emberjs/rfcs/pull/276).

2. Because of the named arguments auto-reflection, it is actually possible
   to configure the `tagName` and classes on the "hidden" component
   instance on the invocation (e.g. `{{foo-bar tagName="footer" class="legalese"}}`).
   This will obviously stop working, but it is also not necessary anymore
   as the component author can simply include the tag in the template.
   Alternatively, the component author can choose to leave out the tag
   and let the caller wrap it in their template.

3. It is possible (but very rare) to configure global injections on the
   component type. Since no component is being instantiated here, those
   properties will not be accessible in the template. 
   
   More broadly, `{{this.foo}}` or the shorthand `{{foo}}` (where it
   would have resolved into a `this` lookup) will always be `undefined`
   (or `null`, perhaps).

## Migration Path

Given the subtle semantics differences enumerated above, it is
not necessarily safe to simply turn on the flag in bigger applications
as it is quite likely that some of the template-only components might be
relying on one or more of these features. Further, removing the wrapper
element might break the layout.

Therefore, the only safe, mechanical transformation is to generate a
JavaScript file for each template-only component (turning them into non-
template only components). We should supplement the change by providing
a codemod that does this for you.

While this would mean that apps would not be able to immediately take
advantage of the feature, it will open the door for new template-only
components to be written in the new semantics.

The user can also audit the components we identified and decide to
delete the JavaScript and migrate them on a case-by-case basis.

The codemod can also come with a more aggressive (and unsound) mode that
simply wraps each template in a `<div>` (to avoid breaking layout in most
cases). This might be acceptable for smaller apps.

For what it's worth, the Ember CLI component blueprint always generate a
JavaScript and a template file, so it might not be that common to find
existing template-only components in an average app.

## Implementation Plan

Finally, for the actual implementation, this would be implemented using
the internal Component Manager API that has already been available for a long
time (and how Curly Components, outlets etc are implemented internally).

It should be very straightforward implementation – essentially just a
Component Manager that requires no capabilities and returns `null` in
`getSelf`.

# How We Teach This

Going forward, the "Outer HTML" semantics will be the default for
template-only components, Glimmer Components and other custom component
types (when the Component Manager API is available), so over time it
should feel quite natural. The experience from the Glimmer experiment
has also proven that this is the more natural programming model for
components.

In the mean time, we still have to deal with the consequence that
existing `Ember.Component` comes with a wrapper element by default. The
mental model for users to understand this is that the `Ember.Component`
class is what is giving you the wrapper element (therefore, template-only
components, which is not an `Ember.Component` does not get one of those).

This should feel quite natual, as the component class is where you
configure the wrapper element (and where you would lookup the API
documentation). You could imagine that the `Ember.Component` is doing
something like this under-the-hood as a convenience feature (which turned
out to be not very convenient after all, but that's a different story):

```js
export const Component = Object.extend({
  tagName: "div",
  classNames: ["ember-view"],

  // This is not real code that exists in the implementation
  render(buffer, template) {
    buffer.append(`<${this.tagName} class="${this.classNames.join(' ')}">`);
    buffer.append(template(this));
    buffer.append(`</${this.tagName}>`);
  }
});
```

# Drawbacks

In general, we avoid flags that puts Ember into very different "modes" as
they causes complication across the whole addon ecosystem. However, as
mentioned above, we don't believe this would be the case here.

# Alternatives

We could keep the current semantics for template-only components. However,
this is usually undesirable, and would only grow to feel more unnatural
as Glimmer Components and friends adopt the "Outer HTML" semantics.

Alternatively, we can make this opt-in per template using a pragma or magic
comment. However, this would be needed for a lot of templates and become
very noisy, and the alternative strategy proposed here (by keeping around
the `Ember.Component` JavaScript file as needed) would be able to accomplish
the same goal with less noise.
