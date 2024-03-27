---
stage: recommended
start-date: 2020-12-22T00:00:00.000Z
release-date: 2021-03-22T00:00:00.000Z
release-versions:
  ember-source: v3.26.0

teams:
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1016 
project-link:
---

# Deprecate using `{{attrs}}` and {{this.attrs}} in templates

## Summary

[RFC #690](https://github.com/emberjs/rfcs/pull/690) had some unfortunate timing where `{{attrs}}` was widely used, but before usage of `{{this.x}}` became widely common. As such, the deprecation messaging [implemented here](https://github.com/emberjs/ember.js/blob/0822efd9c0554f11d01232585596ac70c14a4306/packages/ember-template-compiler/lib/plugins/assert-against-attrs.ts#L58) only messaged in the case of `{{attrs}}`, rather than both `{{attrs}}` and `{{this.attrs}}`. 


The RFC had intended, and even describes in its transition path to move away from `{{this.attrs}}`. [The code for the build time assertion](https://github.com/emberjs/ember.js/blob/0822efd9c0554f11d01232585596ac70c14a4306/packages/ember-template-compiler/lib/plugins/assert-against-attrs.ts#L84) even tried to handle the `{{this.attrs}}` case. 


```hbs
{{attrs.foo}} and {{this.attrs.foo}}

{{! is equivalent to }}
{{@foo}} and {{@foo}}
```

It was a legacy API that existed prior to named arguments being introduced in
Ember, and has continued to be supported via a template transform for some time.

This RFC proposes that we add a deprecation warning to the build-time assertion that forbids `{{attrs}}`, it will also forbid `{{this.attrs}}`. 

## Motivation

The `{{attrs}}` / `{{this.attrs}}` syntax was from a previous iteration of the concepts that
eventually became named argument syntax. Now that named arguments exist in the
framework, and are considered the best practice, there is no reason to continue
supporting this syntax.

## Transition Path

Users who currently rely on referencing `{{attrs}}` or `{{this.attrs}}` can convert their references
to named arguments.

## How We Teach This

### Deprecation Guide

The deprecation guide from [3.26.0](https://deprecations.emberjs.com/v3.x#toc_attrs-arg-access) is already sufficient.

## Alternatives

pre-v6 deprecation.

## Drawbacks

- None
