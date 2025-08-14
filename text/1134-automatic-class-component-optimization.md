---
stage: accepted
start-date: 2025-08-14T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams:
  - framework
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1134
project-link:
suite: 
---

# Automatic class component optimization

## Summary

Enable `@glimmer/component` Component classes with no body (other than the template) to be automatically optimized at build time to the lighter template-only component format, allowing developers to use familiar class syntax with TypeScript generics while maintaining optimal runtime performance.

This will only be available in applications as it will require a decision during compile time based on results of traversing the `template` contents.

## Motivation

Currently, template-only components in Ember are defined using `const` declarations like this:

```glimmer-ts
export const Demo = <template>
  {{@value}}
</template>
```

However, this approach has a significant limitation: **it's not possible to specify generic type parameters** with `const` declarations in TypeScript. This prevents template-only components from having proper type safety when they need to work with generic data types.

For example, consider a template-only component that should work with different types of values:

```glimmer-ts
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

export class Demo<Value> extends Component<ComponentSignature<Value>> {
  <template>
    {{@value}}
  </template>
}
```

However, creating a full component class when you only need a template _feels like_ overkill when folks know that template-only components [are supposed to be more performant][^template-only-faster].

[^template-only-faster]: template-only components are 7 to 26% faster than the `Component` from `@glimmer/component` -- https://nullvoxpopuli.com/2023-12-20-template-only-vs-class-components

## Detailed design

### Build-Time Optimization for Empty Component Classes

This RFC proposes a build-time optimization that automatically compiles `@glimmer/component` Component classes with no body (other than the template) to the template-only component format.

Developers can write components using familiar class syntax with TypeScript generics:

```glimmer-ts
import Component from '@glimmer/component';

interface ComponentSignature<Value> {
  Args: {
    value: Value;
  };
}

export class Demo<Value> extends Component<ComponentSignature<Value>> {
  <template>
    {{@value}}
  </template>
}
```

If the following criteria are met
- no class body members (methods, properties, constructor)
- no `this` identifier within `{{}}` in the template

The `Demo` component would be converted to:

```gjs
export var Demo = template(`@value`);
```

### Why we must parse the template for `this`

Consider:
```gjs
// has no class body members
class A extends Component {
  <template>
     {{! but references `this` and passes it elsewhere}}
     <B @state={{this}} />
  </template>
}

class B extends Component {
  get router() {
    // that component instance from `A` is used here
    return getOwner(this.args.state).lookup('...');
  }
  <template>
    {{this.router.currentURL}}
  </template>
}
```

## How we teach this

This feature introduces **automatic build-time optimization** for Component classes that only contain templates. Developers don't need to learn new APIs or change their mental model - they can continue writing Component classes as they always have, and get automatic performance benefits when appropriate.

### Documentation Updates

#### TypeScript Guide Updates

The [TypeScript: Invokables](https://guides.emberjs.com/release/typescript/core-concepts/invokables/) guide should be updated to include:

1. A new section explaining that Component classes with only templates are automatically optimized
2. Examples showing generic Component classes that benefit from this optimization
3. Guidance on when the optimization applies and when it doesn't

There is little to no benefit to using this feature for JavaScript features unless folks think they might regularly switch between template-only and class components. This could be called out during initial component-authoring guides, but doesn't need much mention, since the goal of this compilation behavior is to be transparent to users (and always correct, so it is not noticed).

#### eslint-plugin-ember

we would disable this lint rule: [no-empty-glimmer-component-classes](https://github.com/ember-cli/eslint-plugin-ember/blob/master/lib/rules/no-empty-glimmer-component-classes.js)

## Drawbacks

## Alternatives

### Alternative 1: New `TemplateOnly` Base Class

Instead of optimizing existing Component classes, introduce a new `TemplateOnly` base class:

```typescript
class Demo<T> extends TemplateOnly<{ Args: { data: T } }> {
  <template>{{@data}}</template>
}
```

See: [RFC #1133](https://github.com/emberjs/rfcs/pull/1133)


## Unresolved questions

n/a