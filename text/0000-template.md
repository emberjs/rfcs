- Start Date: 2019-02-09
- Relevant Teams: Ember.js, Learning
- RFC PR: https://github.com/emberjs/rfcs/pull/443
- Tracking: (leave this empty)

# `trustedHtml` and `trusted-html`

## Summary

This RFC supersedes [RFC 319](https://github.com/emberjs/rfcs/pull/319) and proposes that we:

  * Deprecate `htmlSafe` in favor of a new `trustedHtml` function.
  * Deprecate `{{{` in favor of a new `trusted-html` handlebars helper.

## Motivation

When rendering, Ember escapes values by default. This helps prevent security attack vectors such as XSS when rendering untrusted content from end-users of the application. Sometimes a developer will have a need to bypass this default and render unescaped content directly, for example when rendering sanitized HTML returned from a server endpoint.

Ember currently provides a couple of ways to render unescaped content, [`htmlSafe`](https://www.emberjs.com/api/ember/release/functions/@ember%2Ftemplate/htmlSafe) and the `{{{` handlebars syntax.

There are a number of downsides to these:

  * `htmlSafe` is a confusing name, it's a common and understandable mistake to think that this function sanatizes markup to make it safe.
  * `{{{` is very similar to `{{` and the slight different in syntax doesn't indicate the important difference in how these two syntaxes behave. 
  * `htmlSafe` and `{{{` have similar responsibilities but they have different names.

Replacing `htmlSafe` and `{{{` with `trustedHtml` and `trusted-html` brings a number of benefits:

  * It helps make Ember easier to undertand and more consistent.
  * It helps prevent developers from inadvertently introducing serious security attack vectors by encouraging them to consider the trust aspects of rendering unescaped content.
  * The `html-safe` helper can be used in sub-expressions and the result can be passed around in ways that `{{{` can not.

## Detailed design

### `htmlSafe` -> `trustedHtml`

We'll create a new `trustedHtml` function which will be based on the [existing `htmlSafe` implementation](https://github.com/emberjs/ember.js/blob/dff3c621801999e06dc773ce50a35a97233a0eb5/packages/%40ember/-internals/glimmer/lib/utils/string.ts#L72-L85).

Note: It might be wise to rename `Handlebars.SafeString` to `Handlebars.TrustedString` in future. In the interest of keeping things simple, we'll leave that outside the context of this RFC though.

We'll modify `htmlSafe` to log a deprecation warning and then internally invoke `trustedHtml`. The deprecation warning will be something like:

```
Using `htmlSafe` is deprecated, please use `trustedHtml` instead.
```

### `{{{` -> `trusted-html`

We'll create a new `trusted-html` handlebars helper:

```js
import { helper } from '@ember/component/helper';
import { trustedHtml } from '@ember/string';

export function trustedHtml([html]) {
  return trustedHtml(html);
}
```

We'll create an AST transform in [`packages/ember-template-compiler`](https://github.com/emberjs/ember.js/tree/master/packages/ember-template-compiler) which will emit a deprecation warning for all uses of `{{{`. The deprecation warning will be something like:

```
Using `{{{` is deprecated, please use the `trusted-html` helper instead.
```

We'll create a codemod to help people automatically migrate their uses of `{{{` to `trusted-html`.

## How we teach this

The `trusted-html` template helper should be mentioned in the [API docs](https://emberjs.com/api/ember/release/classes/Ember.Templates.helpers). 

We should replace the current [`htmlSafe` API documentation](https://www.emberjs.com/api/ember/release/functions/@ember%2Ftemplate/htmlSafe) with similar documentation for `trustedHtml`.

If we have a security section in the guides, we should mention how using `trusted-html` and `trustedHtml` requires the developer to understand the risks that rendering unescaped content can pose and that they are asserting that they do trust the content.

We should revisit the binding style attributes warning message and the content that it links to:

```
WARNING: Binding style attributes may introduce cross-site scripting vulnerabilities; please ensure that values being bound are properly escaped. For more information, including how to disable this warning, see https://emberjs.com/deprecations/v1.x/#toc_binding-style-attributes.
```

## Drawbacks

This RFC introduces some API changes which require some changes to existing applications. We'll need to teach Ember users the new function and helper.

## Alternatives

There was some discussion on the [original RFC](https://github.com/emberjs/rfcs/pull/319) for some alternative naming. These include:

 * `trust-this-html`
 * `unescape(d)`
 * `raw`
 * `rawHtml`
 * `dangerously-render-html`
 * `dangerous-html`
 * `unsafe-html`
 * `bypass-sanitization`
 * `bypass-html-escaping`
 * `trusted`

## Unresolved questions

 * Is `trusted-html` the best name? Are there additional suggestions that we should include in the section above?
 * Instead of deprecating `{{{`, perhaps we could just help developers to lint against it?
 * Should these helpers/functions differentiate between HTML and style trusted strings? See [this comment](https://github.com/emberjs/rfcs/pull/443#issuecomment-462282379) for details.
 * Should we remove the current `Binding style attributes may introduce cross-site scripting vulnerabilities` warning? See [this comment](https://github.com/emberjs/rfcs/pull/443#issuecomment-463129055) for more details.
