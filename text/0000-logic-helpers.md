- Start Date: 2016-07-16
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Introduce the following HandleBars helpers to Ember.js:

 - `{{eq}}`
 - `{{not-eq}}`
 - `{{not}}`
 - `{{and}}`
 - `{{or}}`

# Motivation

The proposed helpers are some simple helpers that help adding basic logic to
templates. They are currently already available in an addon called
`[ember-truth-helpers](https://github.com/jmurphyau/ember-truth-helpers)`, but I
believe these helpers are essential, because the `ember-truth-helpers` addon is
one of the most downloaded non Ember CLI default addons.

# Detailed design

See `[ember-truth-helpers](https://github.com/jmurphyau/ember-truth-helpers)`
repo for the detailed design of the helpers.

# How We Teach This

The mandatory documentation should be sufficient to teach this to new users.
Optionally a section in the templates guides could be dedicated to the new logic
helpers.

# Drawbacks

The only drawback I see is that it expands the surface of the API.

# Alternatives

An alternative could be making `ember-truth-helpers` one of the default addons
included when generating a new app/addon.

# Unresolved questions

There are no unresolved questions at this time.
