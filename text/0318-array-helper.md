---
Start Date: 2018-03-24
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/318
Tracking: https://github.com/emberjs/rfc-tracking/issues/23

---

# `array` helper

## Summary

This RFC proposes to add an `array` template helper for creating arrays in templates.

The helper would be invoked as `(array arg1 ... argN)` and return the value `[arg1, ..., argN]`. For example, `(array 'a' 'b' 'c')` would return the value `['a', 'b', 'c']`.


## Motivation

Objects (or hashes) and arrays are the two main data structures in JavaScript. Ember already has a `hash` helper for building objects, so it makes sense to also include an `array` helper for building arrays.

## Detailed design

The design is straightforward and mirrors the design of the `hash` helper. In particular, the important thing to note is that if any of the arguments to the `array` helper change then an entirely new array will be returned, rather than updating the existing array in place.

The implementation would also mirror the [implementation of the `hash` helper](https://github.com/emberjs/ember.js/blob/ec9f4e5e5f4099a77a73bc5a9aa41916f0d15d6d/packages/ember-glimmer/lib/helpers/hash.ts#L49-L51) and would simply capture the positional arguments instead.

## How we teach this

This helper is not an important part of the programming model and can just be mentioned in the [API docs](https://emberjs.com/api/ember/release/classes/Ember.Templates.helpers) like its sibling the `hash` helper.

## Drawbacks

As usual, adding new helpers increases the surface area of the API and file size but in this case it is justified because the file size change is extremely small and its actually filling an existing hole in the API.

## Alternatives

This helper could be left to addons, and indeed there are addons that include this helper. It's also trivial to generate
your own `array` helper with `ember generate helper array`. Humorously, the default helper blueprint generates a helper that already acts like the `array` helper ;)

Nevertheless, I believe it's preferable to include this helper in Ember to fill the hole in Ember's API.
