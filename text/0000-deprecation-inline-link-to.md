- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Deprecate Inline {{link-to}}

## Summary & Motivation

`<LinkTo>` does not support inline invocation, and while `{{link-to}}` does, deprecating the inline form will not only help people migrate but may reduce misuse of the inline arg (what can happen instead of converting to block-from).

Because `{{link-to}}` uses a variety of positional params, it has been inconsistent with itself with respect to the meaning of each position of the params. Sometimes the first param is a route name, sometimes it is the content of the anchor tag.

## Transition Path

Convert all inline `{{link-to}}`s to block-form, or use the `<LinkTo>` component.

## How We Teach This

The ember guides would need to remove mentions of the inline-link form, such as:

https://guides.emberjs.com/release/templates/links/#toc_adding-additional-attributes-on-a-link

A deprecation should be logged in the console whenever an inline `{{link-to}}` is rendered.

Additionally, a linter rule could be added to the template linter to cache usages of inline `{{link-to}}` that aren't rendered during  development.

## Drawbacks

- More people than expected may be using inline link-to

## Alternatives

- We could add a `@text` arg to `<LinkTo>` and just encourage people keep the value of that arg short
