---
Start Date: 2019-09-13
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/375
Tracking: https://github.com/emberjs/rfc-tracking/issues/15

---

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

This RFC proposes that these macros be updated to receive additional dependent
keys via their public API directly via an optional second parameter which is an
array of the keys:

```ts
function filter(filteredPropertyKey: string, callback: Function): ComputedProperty;
function filter(
  filteredPropertyKey: string,
  additionalDependentKeys: string[],
  callback: Function
): ComputedProperty;

function map(mappedPropertyKey: string, callback: Function): ComputedProperty;
function map(
  mappedPropertyKey: string,
  additionalDependentKeys: string[],
  callback: Function
): ComputedProperty;
```

## Deprecation Timeline

The deprecation should follow these steps:

* Update `filter` and `map` to their new APIs
* Add a deprecation warning to uses of `.property` which add dependent keys to
  computed properties.
* Add an optional feature to turn the deprecation into an assertion
* After enough time has passed for addons and users to update, enable the
  optional feature by default in new addons and apps
* Fully remove `.property()` in Ember v4.0.0

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

We could allow additional dependent keys to be passed via an options argument:

```ts
filter(dependentKey: string, options: {
  additionalDependentKeys: [],
  callback: Function,
}): ComputedProperty;

map(dependentKey: string, options: {
  additionalDependentKeys: [],
  callback: Function,
}): ComputedProperty;
```

This is more verbose, but would be very clear.
