- Start Date: 2019-09-13
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Deprecate the computed `.property()` modifier which can be used to add dependent
keys to computed properties.

# Motivation

Currently, computed properties can use the `.property` modifier to add dependent
keys to a computed _after_ the computed has been declared:

```js
foo: computed('strings', {
  get() {
    return this.strings.filter(s => s.includes(this.filterText));
  }
}).property('filterText')
```

In most cases, this ability is redundant, since the dependent keys can be moved
into the original computed declaration and be equivalent:

```js
foo: computed('strings', 'filterText', {
  get() {
    return this.strings.filter(s => s.includes(this.filterText));
  }
})
```

The one exception is in the case of computed _macros_, specifically macros which
accept a _function_ such as `filter()` and `map()`:

```js
foo: filter('strings', function(s) {
  return s.includes(this.filterText);
}).property('filterText')
```

The issue stems from the fact that the inner function can access the class
instance and use dynamic properties from it, and this access is opaque to the
macro.

This API is confusing since it bears a strong resemblance to the older style
of computed property declarations, and at first glance appears to be invalid.
The few edge-case macros where it does legitimately apply can be rewritten to
accept more dependent keys, making it fully redundant.

# Transition Path

As mentioned above, macros which receive a callback function as an argument are
the only valid use of `.property()` in current Ember. Currently, there are two
such macros in Ember core: `map` and `filter`.

This RFC proposes that these macros be updated to receive a list of dependent
keys instead of just one. The first dependent key will be the value to be
mapped/filtered, and the subsequent keys will only invalidate the cache.

```js
foo: filter('strings', 'filterText', function(s) {
  return s.includes(this.filterText);
})

// with ember-decorators
@filter('strings', 'filterText')
foo() {
  return s.includes(this.filterText);
}
```

This will be the recommended approach for macros supplied by addons as well.

## Deprecation Timeline

Once the macros have been updated to accept an arbitrary number of dependent
keys, a deprecation warning should be added to calls to `.property()`. Removing
the functionality will be a breaking change, so it will be set to be removed
in Ember v4.0.0.

# How We Teach This

In most cases, we shouldn't have to teach anything. There are already linting
rules prohibiting `.property()` usage, and the recommended path is to provide
all dependent keys in the original declaration of the computed property. For
users of `map` and `filter` we should ensure that they new documentation is
clear on how to add dependent keys to either macro.

For addon authors that have created their own macros which rely on callbacks and
have similar issues, we should demonstrate how they can structure their macro
API to accept additional dependent keys.

# Drawbacks

The new proposed APIs for `filter` and `map` may be somewhat confusing, since
only the first argument will be filtered/mapped

# Alternatives

We could allow additional dependent keys to be passed to filter/map in a
different way, such as via an options object:

```js
foo: filter(
  'strings',
  {
    dependentKeys: ['filterText']
  },
  function(s) {
    return s.includes(this.filterText);
  }
)

// with ember-decorators
@filter('strings', {
  dependentKeys: ['filterText']
})
foo() {
  return s.includes(this.filterText);
}
```
