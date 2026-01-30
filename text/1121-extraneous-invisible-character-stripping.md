---
stage: recommended
start-date: 2025-07-09T00:00:00.000Z
release-date:
release-versions:
  content-tag: 4.1.0
teams:
  - framework
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1121'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/1127'
  released: 'https://github.com/emberjs/rfcs/pull/1157'
  recommended: 'https://github.com/emberjs/rfcs/pull/1163'
project-link:
suite:
---

# Default Template invisible character Minification

## Summary

Integrate template invisible character minification directly into Ember's template compiler, making the behavior of the ember-hbs-minifier addon available by default for all Ember applications. This will automatically remove unnecessary invisible character from templates while preserving semantic meaning and visual layout, resulting in smaller bundle sizes and improved performance without requiring developer intervention.

## Motivation

HTML templates in Ember applications often contain significant amounts of invisible character that exists purely for developer readability but serves no semantic or visual purpose in the rendered output. This invisible character includes:

- Leading and trailing spaces, tabs, and newlines in text nodes
- Excessive invisible character between elements that gets collapsed by browsers anyway
- Formatting indentation that increases bundle size without benefit
  - This is exacerbated by the adoption of gjs/gts, which has yet an another extra indentation level

Currently, developers must manually install and configure the ember-hbs-minifier addon to achieve optimal template output, and the instructions for doing so in v2 addons and vite apps are not yet available on that repo[^hbs-minifier-maintain]. However, invisible character minification is a widely applicable optimization that benefits virtually all applications and has well-understood safety rules.

[^hbs-minifier-maintain]: ember-hbs-minifier _is_ a third party library, so it can only be as maintained as the maintainers have time for. Here is an issue on their repo for how to [use with a babel config](https://github.com/mainmatter/ember-hbs-minifier/issues/744) (for reference).

The HTML specification clearly defines how browsers handle invisible character:
- Multiple consecutive invisible character characters are collapsed to a single space
- Leading and trailing invisible character in block-level elements is typically ignored
- invisible character-only text nodes between block elements are often not rendered

By understanding these rules, we can safely remove extraneous invisible character while preserving:
- Intentional spacing in inline content
- Preformatted content (like `<pre>` tags)
- Content where invisible character has semantic meaning

Making this optimization available by default will:
1. Reduce bundle sizes for all Ember applications
2. Improve parsing and rendering performance
3. Eliminate the need for developers to discover and configure an additional addon / babebl plugin
4. Provide a consistent, optimized template output across the ecosystem

## Detailed design

Strip the overall indent of a gjs/gts template including the leading / trailing line breaks. -- this is similar to `stripIndent` from [common-tags](https://www.npmjs.com/package/common-tags)

----

It's not possible to safely strip all extraneous invisible characters, as the definition of extraneous is dependent on active CSS.

The governing documentation on these rules are official standards documents:
- [from MDN](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model/Whitespace)
- [from CSS WG: White Space Processing & Control Characters](https://drafts.csswg.org/css-text-3/#white-space-processing)
- [from CSS WG: Collapsible White Space](https://drafts.csswg.org/css-text-3/#collapsible-white-space)


But there is a safe transform that we can do only on gjs/gts
-  The leading and trailing invisible characters for opening `<template>` and closing `</template>` in gjs/gts files should be stripped.
    - This helps determine the "indentation level" which would also be stripped from each line -- effectively de-denting templates to what developers were used to before we started embedding templates in JS/TS.  
    - How to determine indentation:
        - scan all lines after the line with `<template>` and before the line with `</template>`
            - ignore content on the same line as the opening and closing template tags
        - indentation is determined by the smallest number of invisible characters for each line
            - skip lines with 0 characters
            - ideally all other lines have the same or greater indentation
            - if spaces and tabs are mixed, we can ignore tabs -- we don't do any indentation stripping
            - stripping tab indentation can only happen if all lines are indented with tabs


### Examples

> [!NOTE]
> These examples use the format described in [RFC #931](https://github.com/emberjs/rfcs/pull/931) for brevity.

> [!IMPORTANT]
> Every invisible character and line break unneededly adds to the app-compiled wire-format, which leads to more runtime code for end-users to deal with. The more extraneous invisible characters we can eliminate, the better.
> For _aggressive_ invisible character stripping, [ember-hbs-minifier](https://github.com/mainmatter/ember-hbs-minifier/) is still a viable option.

<table>
    <thead>
<th>Input</th>
<th>Output (today)</th>
<th>Output (desired)</th>
</thead>
<tbody>
<tr><td>


```gjs
<template>
  <span>x</span>
</template>
```

</td><td>

```js
export default template(
    '\n  <span>x</span>\n'
);
```

</td><td>

```js
export default template(
    '<span>x</span>'
)
```

</td></tr>
<tr><td>

```gjs
const MyTemplate = <template>
  <span>x</span>
</template>

<template>
  <MyTemplate />
</template>
```

</td><td>

```js
const MyTemplate = template(
    '\n  <span>x</span>\n'
);

export default template(
    '\n  <MyTemplate>\n',
    { scope: () => ({ MyTemplate })}
)
```

</td><td>

```js
const MyTemplate = template(
    '<span>x</span>'
);

export default template(
    '<MyTemplate>',
    { scope: () => ({ MyTemplate })}
)
```

</td></tr>            
<tr><td>


```gjs
<template>
  Hello
  <span>there</span>.
  <p>
    <span>how are you</span>
  </p>
</template>
```

</td><td>

```js
export default template(
    '\n  Hello\n  <span>there</span>.\n  <p>\n    <span>how are you</span>\n  </p>\n'
);
```

</td><td>

```js
export default template(
    'Hello <span>there</span>.\n<p>\n  <span>how are you</span>\n</p>'
)
```

</td></tr>
</tbody>
</table>

### Configuration

There will be no configuration, as this can be enabled with 0 breaking changes to existing behavior.

### Backward Compatibility

- Existing applications using ember-hbs-minifier can continue to do so -- this RFC does not affect the behavior of those transforms.

## How we teach this

This feature requires minimal teaching as it operates transparently. It requires more teaching now to explain all the excessive indentation we have in gjs / gts.

When releasing the next version of `content-tag`, and `ember-template-imports` should note the new behavior.

In the template-tag section of the guide, add a sentence that says that indentation spacing is stripped.

Also, we should add to the guides that to opt-out, folks can add a comment like this:

```gjs
    <template>
{{!-- prevent automatic de-indent --}}
       pre content here
    </template>
```

## Drawbacks

It's _possible_ that someone wrapped their whole component in `white-space: pre`, but we consider the extra indentation behavior a bug from the initial gjs/gts implementation.

## Alternatives

- This RFC could be considered a bugfix.

- Keep ember-hbs-minifier as an optional addon
   - Pros: No breaking changes, opt-in behavior
   - Cons: Many applications miss this optimization, ecosystem fragmentation, we want ember to be a cohesive out-of-the-box framework

- Support opt-in: 
    ```gjs
    <template minifiy>
        <span>x</span>
    </template>
    ```
    Which could alleviate the edge case where folks are using `white-space: pre` on arbitrary contents.

- Breaking change for some CSS use cases: enable all ember-hbs-minifier optimization.

## Unresolved Questions

n/a
