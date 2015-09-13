- Start Date: 2015-09-12
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

This RFC proposes the extraction of a new `url-for` helper from `link-to`.

# Motivation

Decomposing `link-to` into a set of helpers will allow us to simplify its internals while providing new low level primitives for use in performance sensitive applications. `url-for`, the first of these extracted helpers, will allow for simple URL rendering in an Ember application. Extracting separate helpers for dealing with link classes will be suggested in a future RFC.

This proposal would also allow plain URLs to be used within an Ember application without triggering an application reload. For example, a url such as `/about` could be used within a template or helper.

# Detailed design

The `url-for` helper would provide a similar interface to the `link-to` helper in block form:

```
{{#link-to 'photo.comment' 5 primaryComment}}
  Main Comment for the Next Photo
{{/link-to}}
```

The `url-for` equivalent would be:

```
<a href="{{url-for 'photo.comment' 5 primaryComment}}">
  Main Comment for the Next Photo
</a>
```

The `url-for` helper would simply render the URL, no component is needed.

An additional click handler would be setup in [EventDispatcher](https://github.com/emberjs/ember.js/blob/master/packages/ember-views/lib/system/event_dispatcher.js) to capture any clicks on anchors within the ember application. Any anchor URL recognized by the router will be handled by it.

# Drawbacks

This expands the API surface area of Ember.js by introducing a new helper. One additional global event handler would be required.

# Alternatives

The existing [`href-to` addon](https://github.com/intercom/ember-href-to) provides this capability and applications can currently opt-in if they desire.

# Unresolved questions

Is there a better name than `url-to`? What other features of `link-to` should we also support (eg. `replace=true`)?
