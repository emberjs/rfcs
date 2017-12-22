- Start Date: 2017-12-22
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Promote the private API `{{-in-element}}` to public API as `{{in-element}}`.

# Motivation

Developers need to render content out of the regular HTML flow, and `{{-in-element}}` for
that purpose, but it remains private (or _intimate_) API. People is usually wary of using
private APIs (and for good reason) as they may get removed at any time.

If the core team and the community is happy with the current behavior of `{{-in-element}}` it's
time to make it public.

# Detailed design

Create a new `{{in-element}}` as simple alias of `{{-in-element}}` with the exact same signature
and semantics and deprecate the private one.
The deprecated private API will continue to exist until the first LTS release of the 3.X cycle (3.4)
to be finally removed in the next one (3.5).

# How We Teach This

This will be a new build-in helper and must be added to the guides and the API.
For most usages, it will replace some community solution created with the same goal, like
[ember-wormhole](https://github.com/yapplabs/ember-wormhole) or [ember-elsewhere](https://github.com/ef4/ember-elsewhere).
It would be for the best to let the authors of those addons know about this feature so they can
deprecate their packages if they feel there is no longer a need for them, or at least update their
Readme files to let their users know that there is a built-in solution in Ember that might cover
their needs.

# Drawbacks

By augmenting the public API of the framework, the framework is committing to support it for the lifespan
of an entire mayor version (Ember 4.0).

# Alternatives

We can decide that the framework does not want to make public and support this feature, and continue
to rely on community-built addons like we've done until today.

# Unresolved questions

Do we want to make any improvement to `{{-in-element}}` before making it public API?

Some possible ideas:
- Allow to _conditionally_ render the block in place. See https://github.com/DockYard/ember-maybe-in-element
- Allow to receive not only DOM elements as first argument, but also strings, representing the ID of
  other CSS selector.
- Modify or improve the way it behaves during SSR using ember-fastboot.