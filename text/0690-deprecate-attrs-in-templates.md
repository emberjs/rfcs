---
stage: accepted
start-date: 2020-12-22T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/690
project-link:
---

# Deprecate using `{{attrs}}` in templates

## Summary

The `{{attrs}}` object in templates is an alternative way for users to reference
named arguments directly in a template.

```hbs
{{attrs.foo}}

{{! is equivalent to }}
{{@foo}}
```

It was a legacy API that existed prior to named arguments being introduced in
Ember, and has continued to be supported via a template transform for some time.
This RFC proposes that we deprecate this functionality in favor of directly
using named arguments.

## Motivation

The `{{attrs}}` syntax was from a previous iteration of the concepts that
eventually became named argument syntax. Now that named arguments exist in the
framework, and are considered the best practice, there is no reason to continue
supporting this syntax.

## Transition Path

Users who currently rely on referencing `{{attrs}}` can convert their references
to named arguments. This should be highly codemoddable, and we will attempt to
make a codemod to help out with the transition.

## How We Teach This

### Deprecation Guide

The `{{attrs}}` object was an alternative way to reference named arguments in
templates that was introduced prior to named arguments syntax being finalized.
References to properties on `{{attrs}}` can be converted directly to named
argument syntax.

Before:

```hbs
{{attrs.foo}}
{{this.attrs.foo.bar}}
{{deeply (nested attrs.foobar.baz)}}
```

After:

```hbs
 {{@foo}}
 {{@foo.bar}}
 {{deeply (nested @foobar.baz)}}
 ```

## Drawbacks

- None
