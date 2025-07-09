---
stage: accepted
start-date: 2025-07-09T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams:
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1121
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

[^hbs-minifier-maintain]: ember-hbs-minifier _is_ a third party library, so it can only be as maintained as the maintainers have time for

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

The template compiler will include a new AST transform that runs after all user-defined transforms but before the final compilation step. This transform will implement the same invisible character minification logic currently provided by ember-hbs-minifier.

### Core Minification Rules

1. **Text Node Normalization**: Replace leading and trailing invisible character in text nodes with a single space character, unless the text node is entirely invisible character and can be safely removed.

2. **invisible character-Only Node Removal**: Remove text nodes that contain only invisible character characters when they appear:
   - At the beginning or end of element children
   - At the beginning or end of template/block bodies
   - Between block-level elements where they serve no visual purpose

3. **Class Attribute Optimization**: Specifically optimize `class` attributes by:
   - Removing leading and trailing invisible character
   - Collapsing multiple consecutive invisible character characters to single spaces
   - Preserving the optimization across concatenated expressions

### Skip Conditions

The minification will be automatically disabled for elements where invisible character is semantically significant:
   - `<pre>` - Preformatted text
   - `<textarea>` - User input areas where formatting matters
   - `<script>` and `<style>` - Code content

### Examples

This list is non-comprehensive, as we want to adhere to _goals_ rather than place a test suite for this behavior in the RFC

> [!NOTE]
> These examples use the format described in [RFC #931](https://github.com/emberjs/rfcs/pull/931) for brevity.

> [!IMPORTANT]
> Every invisible character and line break unneededly adds to the app-compiled wire-format, which leads to more runtime code for end-users to deal with. The more extraneous invisible characters we can eliminate, the better.

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
</tbody>
</table>

### Configuration

There will be no configuration, as this can be enabled with 0 breaking changes to existing behavior.

### Backward Compatibility

- Existing applications using ember-hbs-minifier can gradually migrate by removing the addon
- The default configuration will match ember-hbs-minifier's behavior
- Applications can disable the feature entirely if needed
- Custom skip configurations from ember-hbs-minifier can be migrated to the new configuration format

### Performance Considerations

- The transform runs during build time, not runtime, so there's no runtime performance impact
- Build time increase is minimal as the transform only processes text nodes and element structure
- Bundle size reduction typically outweighs the small build time cost
- The transform can be cached based on template content

## How we teach this

This feature requires minimal teaching as it operates transparently. 

## Drawbacks

n/a

## Alternatives

Keep ember-hbs-minifier as an optional addon
   - Pros: No breaking changes, opt-in behavior
   - Cons: Many applications miss this optimization, ecosystem fragmentation, we want ember to be a cohesive out-of-the-box framework

## Unresolved Questions

n/a
