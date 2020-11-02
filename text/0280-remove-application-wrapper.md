---
Start Date: 2017-12-11
RFC PR: https://github.com/emberjs/rfcs/pull/280
Ember Issue: https://github.com/emberjs/ember.js/pull/15981

---

# Summary

Introduce a low-level "flag" to remove the automatic wrapper `<div>` around
Ember apps and tests.

# Motivation

In Ember applications today, applications are anchored to some existing HTML
element in the page. Usually, this element is the `<body>` of the document, but it
can be configured to be a different one when the application is defined,
passing a CSS selector to the `rootElement` property:

```js
export default Ember.Application.extend({
  rootElement: '#app'
});
```

However, whatever the root is, the application adds another `<div>` wrapper
that is not required anymore. It's a vestigial remainder of some implementation
detail of how views worked in Ember 1.x. Some sort of wisdom tooth of the original
rendering system that serves no purpose today.

Furthermore, much like a wisdom tooth, it can give us problems. In the past, this element
was configurable using the `ApplicationView`, but when views were removed we lost that
ability. Right now we are stuck with a wrapper element we can't remove nor customize,
which is why some apps target the selector `body > .ember-view` to style this element.

Similarly, in testing there is another `.ember-view` wrapper inside the
`#ember-testing` container for no good reason.

This RFC proposes to add a global flag to remove those wrapper elements,
effectively making the `application.hbs` template have "Outer HTML" semantics, which aligns
well with [the changes recently proposed](https://github.com/emberjs/rfcs/pull/278)
for template-only components, as well as the way Glimmer apps work.

The same flag will also remove the unnecessary extra wrapper inside the testing
container.

# Detailed design

## API Surface

The proposed approach is identical to the one proposed in #278, quoted below:

> We should not expose the flag directly as a public API. Instead, we should
abstract the flag with a "privileged addon" whose only purpose is to enable
the flag. Applications will enable the flag by installing this addon. This
will allow for more flexibility in changing the flag's implementation (the
location, naming, value, or even the existence of it) in the future. From the
user's perspective, it is the addon that provides this functionality. The
flag is simply an internal implementation detail.

> We have done this before in other cases (such as the legacy view addon during
the 2.0 transition period), and it has generally worked well.

> When landing this feature, it will be entirely opt-in for existing apps, but
the Ember CLI application blueprint should be updated to include the addon by
default. At a later time, we should provide another addon that _disables_ the
flag explicitly (installing both addons would be an install-time error). At
that time, we will issue a deprecation warning if the flag is *not set*, with
a message that directs the user to install one of the two addons.

## Migration Path

Given that this change only affects one single point in your application,
I do not believe we need any specific strategy. If the users want to bring
back the wrapper because it breaks their styles or some other reason,
they can just add it manually on the `application.hbs` template, with
any class or id they want.

# How We Teach This

This addon will be opt-in, but at some point it will become part of
the default blueprint. This change, rather than introducing a new concept, *removes*
an old one. Users won't have to google what is the way to remove or customize
the implicit application wrapper of the app (to sadly discover that is not
even possible), but instead they will add a wrapper only if they want,
and in the same way they would add a wrapper in any other point of their application,
with regular Handlebars.

# Drawbacks

There is a possibility that removing the wrapper can break styles for some apps,
but since adding the wrapper back is just editing the `application.hbs` template,
that is probably a minor drawback.

There is also a non-zero chance that some testing addon is relying on the `#ember-testing > .ember-view`
HTML hierarchy for some reason, and those addons would have to be updated.

# Alternatives

Leave things as they are today.
