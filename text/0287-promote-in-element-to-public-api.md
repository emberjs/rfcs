---
Start Date: 2017-12-22
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/287
Tracking: https://github.com/emberjs/rfc-tracking/issues/25

---

# Summary

Promote the private API `{{-in-element}}` to public API as `{{in-element}}`.

# Motivation

Sometimes developers need to render content out of the regular HTML flow. This concept is often also
called "portals". Some components like dropdowns and modals use this technique to render stuff close
to the root of the page to bypass CSS overflow rules. Some apps that are embedded into static pages
even use this technique to update parts of the page **outside** the app itself.

This need need has been covered by solutions developed in the user space but it was so common that
glimmer baked it into the VM in the form of `{{-in-element}}`, but it remains private (or _intimate_) API.
People is usually wary of using private APIs (and for good reason) as they may get removed at any time.

If the core team and the community is happy with the current behavior of `{{-in-element}}` it's
time to make it public.

# Detailed design

The existing API of `{{-in-element}}` is very simple:

* It takes a single positional param `destinationElement` that is a DOM element, and a block.
* The given block is rendered not where it is located, but inside the given `destination` element, at
the end of it if there is any other content on the destination element.
* If `destinationElement` is null/undefined then it doesn't render anything but it doesn't error.
* If `destinationElement` is false/0/"" it raises an assertion in development but fails silently in production.
* If `destinationElement` changes the block is removed from the previous destination and added to the new one. This
process tears down the rendered content on the initial destination and renders it again on the new one, meaning
that any component withing the block will be destroyed and instantiated again (calling the appropiate lifecycle hooks),
so transient HTML state like the value of an input will be lost unless manually preserved somewhere else, like a service.
* If the destination element is an invalid value (a string, a number ...) it throws an `parent.insertBefore is not a function` error. I think
that throwing an error is correct but the error message could be improved.
* If the destination element has a different context (like SVG) the content will be appended normally by the glimmer VM,
which doesn't try to validate the correctness of the generated HTML. This is normal behavior in Glimmer, not
an exception, and users must be aware that rendering invalid markup might be interpreted or auto-corrected in
unexpected ways by the browser when in SSR mode.
* Rendering into a foreign object (an element within an `<iframe>`) should be disallowed initially. If someone
asks for this feature it would require an RFC to explore the consequences.

Example usage:

```hbs
{{#-in-element destinationElement}}
  <div>Some content</div>
{{/-in-element}}
```

The current implementation only suggests creating a new `{{in-element}}` construct that is a simple
alias of `{{-in-element}}` with the exact same params and behavior, and then, after a while, remove
the private one.

Although `{{-in-element}}` is technically private, there there is enough people using it to deserve
a deprecation. I suggest keeping the deprecated private API will until the first LTS release of the
3.X cycle (3.4) to be finally removed in the next one (3.5).

### Small proposed changes

There is however one part of the behavior that the core team wants to make explicit before promoting
the private API to public, and that is how the content is added to the destination when there is other
content already there.

The desired behavior is that, by default, the rendered content will **replace all the content of the destination**,
effectively becoming the its `innerHTML`.
In the current behaviour the rendered content is appended as the end of any existing content. This will still
be supported by passing `insertBefore=null`, but it will not be the default anymore.
Any other value passed to `insertBefore` must produce an error.


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
