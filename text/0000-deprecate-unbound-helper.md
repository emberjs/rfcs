---
Stage: Accepted
Start Date: 2021-03-13
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: 
---

# Deprecate `{{unbound}}` Helper

## Summary

This RFC proposes to deprecate the `{{unbound}}` template helper.

## Motivation

Taken from the [`{{unbound}}` helper's docs](https://api.emberjs.com/ember/3.25/classes/Ember.Templates.helpers/methods/unbound?anchor=unbound):
> The `{{unbound}}` helper disconnects the one-way binding of a property, essentially freezing its value at the moment of rendering.

In other words, only the initial value is rendered/returned, even if the property is updated afterwards.

The `{{unbound}}` helper was mostly used to address performance related issues, which should now be a thing of the past. This was mentioned in the guides for the [block and multi-argument deprecation](https://deprecations.emberjs.com/v1.x#toc_block-and-multi-argument-unbound-helper).

There's also an [`ember-template-lint` rule](https://github.com/ember-template-lint/ember-template-lint/blob/master/docs/rule/no-unbound.md) which specifically recommends to not use the `{{unbound}}` helper anymore, because it no longer offers any performance benefits, and it's also considered a bad practice to only render the initial value of a property which could potentially be updated later on.

## Transition Path

Users who are currently using the `{{unbound}}` helper for performance benefits, should be able to remove the helper without any other needed changes.

Users who are currently using the `{{unbound}}` helper for its freeze functionality (because they are only interested in the initial value) need to update their code to provide this functionality themselves.

### Example

#### Before

```hbs
{{! app/templates/user.hbs }}

<User @name={{this.user.name}}>
```

```hbs
{{! app/components/user.hbs }}

{{unbound @name}}
```

#### After

```hbs
{{! app/templates/user.hbs }}

<User @name={{this.user.name}}>
```

```js
// app/components/user.js

import Component from '@glimmer/component';

export default class UserComponent extends Component {
  name = this.args.name;
}
```

```hbs
{{! app/components/user.hbs }}

{{this.name}}
```

## How We Teach This

As far as I can tell the Ember guides do not need to be updated to reflect this deprecation (or possible future removal) as they do not include any usage of the `{{unbound}}` helper.

The `{{unbound}}` helper is only documented in the API documentation.

## Drawbacks

Quite a few deprecations have already been introduced in the 3.X cycle. This of course, will only add to that. App creators and addon authors will have to update their projects accordingly.

A code search on [Ember Observer](https://emberobserver.com/) returns the following results:
1. [Searching for `{{unbound` in every addon's `addon` folder](https://emberobserver.com/code-search?codeQuery=%7B%7Bunbound&fileFilter=addon&sort=score&sortAscending=false)
2. [Searching for `(unbound` in every addon's `addon` folder](https://emberobserver.com/code-search?codeQuery=%28unbound&fileFilter=addon&sort=score&sortAscending=false)

The first search seems to be the most accurate as the second search also returns a lot of matches in JavaScript files.

At first glance, most addons seem to use the `{{unbound}}` helper for its freeze functionality, which can be addressed using a similar approach to the one suggested in the [Transition Path - Example](#example) section.

More notably, [vertical-collection](https://github.com/html-next/vertical-collection) uses the `{{unbound}}` helper as well. I assume `vertical-collection` does (did?) mainly use the `{{unbound}}` helper for performance reasons. At this moment, it's unclear to me what the alternative approach would be in this case.

## Alternatives

- Not deprecate the `{{unbound}}` helper and keep supporting it
- Provide the `{{unbound}}` helper as an Ember addon

## Unresolved questions

- None at the moment