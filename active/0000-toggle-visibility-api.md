- Start Date: 2014-12-24
- RFC PR:
- Ember Issue:

# Summary

Expose as public API the places where views set or toggle their visibility.

# Motivation

In the current implementation, Ember Views set initial visibility via
`buffer.style('display', 'none')` and toggle it via
`$el.toggle(isVisible);`. That is, the implementation uses `visibility`.
HTML5 introduced a new `hidden` attribute for visibility toggling. The primary
intent is provide a more semantic marker for *hiddenness*. As an added
benefit, the new attribute also lets designers style `hidden` elements how
they want, including the possibility of transitions.

# Detailed design

 1. Extract a new helper, `Ember.View.prototype._applyVisibilityToBuffer`,
    from `Ember.View.prototype.applyAttributesToBuffer`. By default, this
    helper will perform the existing logic:
    `buffer.style('display', 'none')`
 2. Extract a new helper, `Ember.View.prototype._toggleElementVisibility`,
    from `Ember.View.prototype._toggleVisibility`. By default, this helper
    will perform the existing logic:
    `$el.toggle(isVisible);`

# Drawbacks

 * It increases the API surface area of `Ember.View`
 * The extra function calls negatively impact performance, even if only slightly

# Alternatives

 1. Clients could monkey-patch `applyAttributesToBuffer` and
    `_toggleVisibility`. These methods are fairly long, though, and
    would have to be copied wholesale.
 2. `Ember.View` could switch to `hidden` by default. Not all
    browsers have sensible default behavior for this attribute, so
    if Ember goes this route, it should probably also inject a
    style like `[hidden] { display: none; }` that works welll,
    but is easily overridden.

# Unresolved questions

 * Is `_toggleElementVisibility` intent-revealing? Its name doesn't
   suggest an *obvious* difference from `_toggleVisibility`.
