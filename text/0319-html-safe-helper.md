- Start Date: 2018-03-24
- RFC PR: https://github.com/emberjs/rfcs/pull/319
- Ember Issue: (leave this empty)

# `html-safe` helper

## Summary

This RFC proposes to add an `html-safe` template helper for creating `SafeString`s in templates.

The helper would be invoked as `(html-safe trustedHTML)` and return a `SafeString` that wraps the string of trusted HTML using the `htmlSafe` function in `@ember/string`.

## Motivation

Ember should encourage people to use `SafeString` and one way to do that is to make them easier to use. Sometimes it can be inconvenient to use the `htmlSafe` function because it would require you to create a component when you otherwise wouldn't need one. This is especially common inside of {{#each}} blocks. If we had a `html-safe` helper than you could express this in the template.

## Detailed design

The `html-safe` helper will be implemented similar to other helpers:

```js
import { helper } from '@ember/component/helper';
import { htmlSafe } from '@ember/string';

export function htmlSafe([trustedHTML]) {
  return htmlSafe(trustedHTML);
}
```

It's worth noting that the `htmlSafe` function already handles edge cases like `null` and `undefined` gracefully so it works well as a Handlebars helper.

## How we teach this

This helper should be mentioned in the [API docs](https://emberjs.com/api/ember/release/classes/Ember.Templates.helpers). I couldn't find a security section in the guides, but if it exists it should also go there. The warning message that you get when you use a non-SafeString in an insecure place could also mention this helper.

## Drawbacks

As usual, adding new helpers increases the surface area of the API and file size but in this case it is justified because the file size change is extremely small and its making it easier to build secure apps.

## Alternatives

While it's very easy to write this helper yourself, it costs us very little to provide it in Ember ever so slightly reduces the barrier to writing more secure apps.
