---
Start Date: 2015-04-09
RFC PR: https://github.com/emberjs/rfcs/pull/46
Ember Issue: https://github.com/emberjs/ember.js/pull/11440

---

# Summary

Fully encapsulate and privatize the `Container` and `Registry` classes by
exposing a select subset of public methods on `Application` and
`ApplicationInstance`.

# Motivation

The `Container` and `Registry` classes currently lead a confusing life of
semi-private exclusion within Ember applications. They are undocumented
publicly but not fully private either, as knowledge of their particulars is
required for developing both initializers and unit tests. This situation has
become untenable as the new `Registry` class has been extracted from
`Container`, and the complexity of their usage has grown across
`Application` and `ApplicationInstance` classes.

We can bring sanity to this situation by continuing the work started at the
`Application` level to expose methods such as `register` and `inject` from the
internally maintained `Registry`.

Furthermore, once `Container` and `Registry` are fully private, their
architecture and documentation can be cleaned up. For instance, a
`Container` can freely reference its associated `Registry` as `registry`
rather than `_registry`, as it can be assumed that only framework developers
will reference this property.

# Detailed design

`Application` will expose the following methods from its internally maintained
registry:

* `register`
* `inject`
* `registerOptions` - mapped to `Registry#options`
* `registerOptionsForType` - mapped to `Registry#optionsForType`

`ApplicationInstance` will also expose the the same methods. However, these
methods will be exposed from its own internally maintained registry, which
has the associated `Application`'s registry configured as a "fall back". No
direct path will be provided from the `ApplicationInstance` to the
`Application`'s registry.

`ApplicationInstance` will also expose the following methods from its
internally maintained container:

* `lookup`
* `lookupFactory`

`ApplicationInstance` will cease exposing `container`, `registry`, and
`applicationRegistry` publicly.

`Application` initializers will receive a single argument to `initialize`:
`application`.

Likewise, `ApplicationInstance` initializers will receive a single argument
to `initialize`: `applicationInstance`.

`Container` and `Registry` will be made fully private and documented as
such. Each `Container` will freely reference its associated `Registry` as
`registry` rather than `_registry`.

[ember-test-helpers](https://github.com/switchfly/ember-test-helpers)
will provide an `isolatedApplicationInstance` method instead of an
`isolatedContainer` for unit testing. A mechanism will be developed to specify
which initializers should be engaged in the initialization of this instance.
In this way, we can avoid duplication of registration logic, as is currently
done in a most un-DRY manner in the [isolatedContainer](https://github.com/switchfly/ember-test-helpers/blob/master/lib/ember-test-helpers/isolated-container.js#L56-L79).

# Drawbacks

This refactor will require maintaining backwards compatibility and
deprecation warnings until Ember 2.0. This will temporarily increase
internal code complexity and file sizes.

# Alternatives

The obvious alternative is to make `Container` and `Registry` fully public
and documented. An application's registry would be available as a `registry`
property. An application instance's container would remain available as
`container`.

We could still pass an `Application` into application initializers
and an `ApplicationInstance` into application instance initializers.

If this alternative is taken, I would suggest that `Application` should
deprecate `register` and `inject` in favor of calling the equivalents on its
public `registry`.

Regardless of which alternative is chosen, we should ensure that the public
aspects of container and registry usage are well documented.

# Unresolved questions

* Are the public methods listed above sufficient or should any others be
exposed?

* What mechanism should be used to engage initializers in unit and
integration tests? Should test modules simply have an `initializers` array,
similar to the current `needs` array?

* Given the semi-private nature of containers and registries, we may not need
to worry about semver for deprecations. However, we should be good citizens
and properly deprecate as much as possible. Some real world use cases in
initializers will no doubt be a surprise, so we need to tread carefully.
