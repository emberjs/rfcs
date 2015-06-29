- Start Date: 2015-06-29
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Add ability to lookup components and helpers directly from within a
template by using `:` to namespace the tag.

Example:

```hbs
<!-- curly -->
{{foo:bar}}

<!-- angle -->
<foo:bar></foo:bar>
```

This syntax is valid HTML, works with Glimmer, and the curly version
works as well.

I also propose that namespaced components *do not* need to adhere to the
rule of hyphenation. The spirit of that rule is to avoid components that
could collide with pre-existing HTML tags. With the namespacing this
would be avoided and the hyphenation shouldn't be necessary.

# Motivation

The current path to using addon components and helpers is to
import/export them from within your own app's namespace. In most cases
this just ends up being an empty component or helper that does nothing by
provide access. Allowing addon components and helpers to be called from
their original namespace will free application developers from this
ceremony. In addition, it could reduce the burden on addon developers if
there are any bugs introduced by the application developer during the
import/export process currently used.

# Detailed design

The HTMLBars `lookup-helpers` and Ember Views' `component_lookup` need
to be modified to parse this namespace.

`foo:bar` => `foo@helper:bar`
`foo:bar` => `foo@component:bar`

The current order of attempting to find the helper first then falling
back to the component will be respected.

# Drawbacks

This could introduce confusion around Ember's own module path syntax as
that uses `@`s to denote namespaces.

# Alternatives

Ideally we would have used `@` for the namespace:

`<foo@bar></foo@bar>`

but this syntax is not valid HTML and thus does not work with Glimmer
(today) and HTMLBars cannot parse it. The `:` syntax proposed in this
RFC does not require any changes to the current parser.

# Unresolved questions

There are probably a few different places that the namespaced components
should also be accessible. For example with the `component` helper:

```hbs
{{component "foo:bar"}}
```

However, at this point do we continue the `:` version or also allow the
`@` version?

Ember Inspector doesn't produce the correct meta-data for namespaced
components:

![](http://i.imgur.com/ITDVfX4.png)
