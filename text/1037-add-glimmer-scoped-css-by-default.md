---
stage: accepted
start-date: 2024-07-09T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams:
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1037
project-link:
suite: 
---

# Make Scoped CSS the default in 'template-tags'

## Summary

A component is defined as a unit of UI with its action and style. 

```gjs
<template>
  <h1>Scoped CSS</h1>
    <p>How is scoped done in Emberjs template-tag?</p>

  <style>
      p { font-family: Roboto};
    </style>
</template>
```
Based on my observation while using `template-tags` to author
components, the style tag inside `template-tags` is global, which I think
should not be because it may affect other components. Also, libraries or
imported components might have used the same style name, so the issue will become nested
and, therefore won't be easy to trace. Even if one works around this using a
unique class wrapper for each component, the possibility of collision is still
present, and uniqueness is not guaranteed project-wide.


## Motivation

To make `template-tags` fulfil all three things (UI, action, style) that
make an excellent component authoring tool. This will make the authoring of
components with `template-tags` much better and safer.

Making scoped-CSS the default will improve the developer experience by not having
to fight spill CSS that is affecting other components.


## Detailed design

There is an implementation already.  [glimmer scoped-css](https://github.com/cardstack/glimmer-scoped-css)


