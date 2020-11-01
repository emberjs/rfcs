---
Start Date: 2016/12/05
RFC PR: https://github.com/emberjs/rfcs/pull/186
Ember Issue: (leave this empty)

---

# Summary

Track unique history location states

# Motivation

The path alone does not provide enough information. For example, if you
visit page A, scroll down, then click on a link to page B, then click on
a link back to page A. Your actual browser history stack is [A, B, A].
Each of those nodes in the history should have their own unique scroll
position. In order to record this position we need a UUID 
for each node in the history.

This API will allow other libraries to reflect upon each location to
determine unique state. For example,
[ember-router-scroll](https://github.com/dollarshaveclub/ember-router-scroll)
is making use of a [modified `Ember.HistoryLocation` object to get this
behavior](https://github.com/dollarshaveclub/ember-router-scroll/blob/master/addon/locations/router-scroll.js).

Tracking unique state is required when setting the scroll position
properly based upon where you are in the history stack, as described in
[Motivation](#motivation)

# Detailed design

Code: [PR#14011](https://github.com/emberjs/ember.js/pull/14011)

We simply unique identifier (UUID) so we can track uniqueness on two
dimensions. Both `path` and the generated `uuid`. A simple UUID
generator such as
https://gist.github.com/lukemelia/9daf074b1b2dfebc0bd87552d0f6a537
should suffice.

# How We Teach This

We could describe what meta data is generated for each location in the
history stack. For example, it could look like:

```js
// visit page A

[
  { path: '/', uuid: 1 }
]

// visit page B

[
  { path: '/about', uuid: 2 },
  { path: '/', uuid: 1 }
]

// visit page A

[
  { path: '/', uuid: 3 },
  { path: '/about', uuid: 2 },
  { path: '/', uuid: 1 }
]

// click back button

[
  { path: '/about', uuid: 2 },
  { path: '/', uuid: 1 }
]
```

# Drawbacks

* The property access is writable

# Alternatives

The purpose for this behavior is to enable scroll position libraries.
There are two other solutions in the wild. One is in the guides that
suggests resetting the scroll position to `(0, 0)` on each new route
entry. The other is
[ember-memory-scroll](https://github.com/ef4/memory-scroll) which I
believe is better suited for tracking scroll positions for components
rather than the current page.

However, in both cases neither solution provides the experience that
users have come to expect from server-rendered pages. The browser tracks
scroll position and restores it when you revisit the page in the history
stack. The scroll position is unique even if you have multiple instances
of the same page in the stack.

# Unresolved questions

None at this time.
