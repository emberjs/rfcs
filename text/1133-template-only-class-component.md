---
stage: accepted
start-date: 2025-08-14T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams:
  - framework
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1133
project-link:
suite: 
---

# Template-Only Class Components

## Summary

Add a new `TemplateOnly` base class to `@glimmer/component` that enables template-only components to be defined as classes, allowing them to support TypeScript generics for better type safety.

## Motivation

Currently, template-only components in Ember are defined using `const` declarations like this:

```typescript
export const Demo = <template>
  {{@value}}
</template>
```

However, this approach has a significant limitation: **it's not possible to specify generic type parameters** with `const` declarations in TypeScript. This prevents template-only components from having proper type safety when they need to work with generic data types.

For example, consider a template-only component that should work with different types of values:

```typescript
import type { TOC } from '@ember/component/template-only';

// This doesn't work - can't add generics to const declarations
export const Demo<Value>: TOC<{ // ❌ Syntax error
  Args: {
    value: Value
  }
}> = <template>  
  {{@value}}
</template>
```

This limitation forces developers to either:
1. Give up type safety and use `any` or `unknown` for component arguments
2. Create unnecessary `@glimmer/component` Component classes to add TypeScript generics -- [these are slower][^template-only-faster]
3. Create multiple duplicate components for different types

Class-based components don't have the syntax limitation:

```glimmer-ts
interface ComponentSignature<Value> {
  Args: {
    value: Value;
  };
}

class Demo<Value> extends Component<ComponentSignature<Value>> {
  <template>
    {{@value}}
  </template>
}
```

Creating a full component class when you only need a template feels like overkill and goes against the principle of template-only components [being lightweight][^template-only-faster].

[^template-only-faster]: template-only components are 7 to 26% faster than the `Component` from `@glimmer/component` -- https://nullvoxpopuli.com/2023-12-20-template-only-vs-class-components

## Detailed design

### New `TemplateOnly` Export

This RFC proposes adding a new `TemplateOnly` export to `@glimmer/component` that can be used as a base class for template-only components:

```glimmer-ts
import { TemplateOnly } from '@glimmer/component';

class Demo<Value> extends TemplateOnly<{
  Args: {
    value: Value;
  };
}> {
  <template>
    {{@value}}
  </template>
}

export default Demo;
```

At runtime, this is _functionally_ the same as the `const` template-only components today.
In that the `TemplateOnly` class we re-export from `@glimmer/component` would be associated with a light-weight component manager, as template-only components are today.

There is no `this`, so there will need to be a lint that forbids use of `this` in `TemplateOnly` classes.

### Compilation

Classes have a feature similar to functions, which is useful for defining more than one in the same file: hoisting.

If we compiled to the `const` representation that we have today, we'd have a disagreement between editor expectations and runtime behavior.

So compilation of these components should use `var` instead of `const`.

Otherwise, all other aspects can be the same as existing compilation today:

```glimmer-ts
// Source
class Demo<Value> extends TemplateOnly<{
  Args: { value: Value };
}> {
  <template>
    {{@value}}
  </template>
}

// Compiled output (same as current template-only components)
import { precompileTemplate } from '@ember/template-compilation';
import { setComponentTemplate } from '@ember/component';
import templateOnlyComponent from '@ember/component/template-only';

export default setComponentTemplate(
  precompileTemplate(`{{@value}}`, {
    strictMode: true,
    scope: () => ({}),
  }),
  templateOnlyComponent()
);
```

Existing babel-based compnoent compilation occurs in [`babel-plugin-ember-template-compilation`](https://github.com/emberjs/babel-plugin-ember-template-compilation).
But the `<template>` transforms occur in [`content-tag`](https://github.com/embroider-build/content-tag), and `content-tag`'s output would be:
```typescript
class Demo<Value> extends TemplateOnly<{
  Args: { value: Value };
}> {
  static {
    template(`{{@value}}`, { component: this, scope: () => ({}) })
  }
}
```
So `babel-plugin-ember-template-compilation` could optimze this to:
```javascript
var Demo = template(`{{@value}}`, { scope: () => ({}) });
```

However, because we also support compile-less environments, the `TemplateOnly` export would _work_ if it made it to the runtime.

[Here is an example implementation of that fallback behavior in the Limber REPL]()

### Usage Examples

#### Simple Generic Component

```typescript
import { TemplateOnly } from '@glimmer/component';

class ItemDisplay<T> extends TemplateOnly<{
  Args: {
    item: T;
    displayProperty: keyof T;
  };
}> {
  <template>
    <div class="item">
      {{get @item @displayProperty}}
    </div>
  </template>
}
```

#### Component with Blocks

```typescript
import { TemplateOnly } from '@glimmer/component';

class List<T> extends TemplateOnly<{
  Args: {
    items: T[];
  };
  Blocks: {
    default: [item: T, index: number];
    empty: [];
  };
}> {
  <template>
    {{#if @items.length}}
      <ul>
        {{#each @items as |item index|}}
          <li>{{yield item index}}</li>
        {{/each}}
      </ul>
    {{else}}
      {{yield to="empty"}}
    {{/if}}
  </template>
}
```


## How we teach this

The feature introduces one new concept: **Template-Only Class Components**. These are components that use class syntax purely for TypeScript generic support while maintaining template-only semantics at runtime.

> [!IMPORTANT]
> More importantly though, our eslint plugin, should have a way to automatically flip between the component formats to reduce the amount of shuffling that developers have to do when they need to switch formats -- this may also make sense as a glint code-action.

### Documentation Updates

#### TypeScript Guide Updates

The [TypeScript: Invokables](https://guides.emberjs.com/release/typescript/core-concepts/invokables/) guide should be updated to include:

1. A new section explaining when and how to use `TemplateOnly` for generic template-only components
2. Examples showing the above examples using generics
3. Guidance on choosing between `const` and `class extends TemplateOnly`

### Migration Path

_There is no need for migration_.

This feature is purely additive and doesn't affect existing template-only components. Developers can continue (and likely should prefer) using the current `const` syntax for simple cases and opt into the class syntax when they need generics.

```typescript
// Existing syntax still works
const SimpleComponent = <template>
  <p>Hello {{@name}}</p>
</template>

// New syntax for when you need generics
class GenericComponent<T> extends TemplateOnly<{
  Args: { data: T };
}> {
  <template>
    <pre>{{json-stringify @data}}</pre>
  </template>
}
```

## Drawbacks

### Learning Complexity

Adding another way to define template-only components could be confusing for new users. However, this is mitigated by:
- The feature is TypeScript-specific and optional
- It follows familiar class extension patterns
- Clear documentation about when to use each approach
- Eslint and Glint tooling can reduce the switching costs (which we probably want anyway for our existing component format switching)

## Alternatives

n/a

## Unresolved questions

n/a