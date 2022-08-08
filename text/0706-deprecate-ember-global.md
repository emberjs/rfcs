---
stage: accepted
start-date: 2021-02-01T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/706
project-link:
---

# Deprecate the Ember Global

## Summary

Deprecate the Ember global, `window.Ember`, and replace it fully with
`import Ember from 'ember';`

## Motivation

The Ember global, available at `window.Ember` (or `globalThis.Ember` in all
environments) is a mostly legacy API from before Ember adopted native ES
modules. Modern Ember apps do not use the global, and in general it is mostly
used by legacy code and occasionally in examples. However, it still exists and
remains undeprecated.

The primary motivation for deprecating the global object is for _tree shaking_
purposes. Because the global object is assigned eagerly to `window.Ember`
whenever Ember is loaded, it essentially means that every Ember application
implicitly uses every single API that is exported on the global object. This
means that we cannot tree shake any of those APIs, and we must in fact load them
all eagerly and execute all of their code as soon as Ember is loaded.

While it is tempting to remove the `Ember` object altogether, there are a number
of undeprecated legacy and intimate APIs that are only available through this
object, such os `Ember.meta`. This is why this RFC proposes only deprecating
accessing `Ember` _via_ `window.Ember`/`globalThis.Ember`. Instead, users will
have to import `Ember` using standard ES module syntax if they want to use it.

```js
import Ember from 'ember';
```

This ensures there is an _explicit_ dependency if it is used, and which will
allow us to treeshake in the future if nothing imports `Ember`. In time, we will
be able to deprecate the `Ember` object altogether.

## Transition Path

When the `Ember` object is defined on the global, we will create a getter for it
that also issues a deprecation. Users who currently use the global will have to
add `import Ember from 'ember';` at the top of the files in which they use it.

## How we teach this

### Deprecation Guide

Accessing Ember on the global context (e.g. `window.Ember`, `globalThis.Ember`,
or just `Ember` without importing it) is no longer supported. Migrate to
importing Ember explicitly instead.

Before:

```js
export default class MyComponent extends Ember.Component {
  // ...
}
```

After:

```js
import Ember from 'ember';

export default class MyComponent extends Ember.Component {
  // ...
}
```

Alternatively, consider converting to use the Ember modules API equivalent to
the API you are using:

```js
import Component from '@ember/component';

export default class MyComponent extends Component {
  // ...
}
```

If there is no modules API equivalent, consider refactoring away from using that
API.

## Drawbacks

- Introduces churn in legacy codebases

- Some polyfills and addons use `Ember` in places where the modules API is not
  available, such as in `vendor` files. These are advanced use cases in general,
  and there are possible workarounds (such as using the `require` global
  function), so this shouldn't block us from removing the global.

## Alternatives

- Keep the global indefinitely.
