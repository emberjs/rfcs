---
stage: accepted
start-date: 2026-05-29T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1196
project-link:
suite:
---

# Assert on invalid usage of `{{outlet}}`

## Summary

This RFC proposes that invalid uses of `{{outlet}}` should fail explicitly with a clear assertion.

This proposal addresses emberjs/ember.js#18547 and corresponds to the implementation proposed in emberjs/ember.js#21422.

`{{outlet}}` is a routing primitive. It should only be valid in routing contexts where Ember can render child route templates. Using `{{outlet}}` in unsupported contexts, such as component templates or non-routable engine templates, should produce a clear diagnostic instead of silently doing the wrong thing or failing later in a confusing way.

## Motivation

`{{outlet}}` marks where child routes render their templates. Because of that, it only has meaningful behavior when used in a routing context.

Today, it is possible to write `{{outlet}}` in places where that routing behavior is not available. For example, an app can accidentally place `{{outlet}}` inside a component template. Components are not part of the route hierarchy and do not have child routes that can render into them, so there is no routing outlet for Ember to fill in that context.

The resulting behavior is not clear to the developer.

It is also possible for a non-routable engine mounted with `{{mount}}` to have `{{outlet}}` in its application template, even though that engine does not have a router and cannot provide routing outlet behavior.

For users, these mistakes can be hard to understand. They may see confusing behavior, delayed failures, or no obvious explanation of why the outlet is not working.

Ember should give a clear error when a framework primitive is used in a place where it cannot work. This RFC proposes making invalid `{{outlet}}` usage fail explicitly with a useful assertion message.

The expected outcome is:

- users get a clear error when `{{outlet}}` is used incorrectly;
- the routing-only nature of `{{outlet}}` becomes easier to understand;
- Ember avoids silently accepting unsupported template patterns;
- apps and addons can migrate invalid usage to route templates or component composition patterns.

## Detailed design

Ember should detect invalid uses of `{{outlet}}` and throw an assertion with a clear error message.

The assertion should apply when `{{outlet}}` is used in a template where Ember does not provide a valid routing outlet context.

This RFC specifically covers two invalid cases:

1. `{{outlet}}` used in a component template.
2. `{{outlet}}` used in the application template of a non-routable engine mounted with `{{mount}}`.

Invalid usage in a component template:

```gjs
// app/components/navigation-shell.gjs

<template>
  <header>
    Navigation
  </header>

  {{outlet}}
</template>
```

This should assert because component templates do not provide a route outlet context.

Invalid usage in the application template of a non-routable engine:

```hbs
{{! lib/my-engine/addon/templates/application.hbs }}

<section>
  {{outlet}}
</section>
```

This should assert when the engine is mounted with `{{mount}}` and does not have a router.

Valid usage in route templates:

```gjs
// app/templates/application.gjs

<template>
  <nav>
    ...
  </nav>

  {{outlet}}
</template>
```

```gjs
// app/templates/posts.gjs

<template>
  <h1>Posts</h1>

  {{outlet}}
</template>
```

In these examples, `{{outlet}}` is valid because the templates participate in routing and can render child route templates.

The assertion message should explain both what went wrong and how to fix it. The current implementation uses this message:

```txt
`{{outlet}}` may only be used in route templates. It cannot be used in component templates or non-routable engine templates.
```

The proposed implementation performs this check at runtime in the `{{outlet}}` helper.

The check rejects `{{outlet}}` when either of the following is true:

1. the current template scope does not provide `outletState`;
2. the current owner belongs to an engine parent, but that engine is not routable.

This matches the two invalid cases covered by this RFC: component templates and non-routable engine templates.

The runtime check is intentional. It uses information that is available when `{{outlet}}` is evaluated, including the current template scope and whether the current owner belongs to a non-routable engine.

This proposal does not require the exact assertion wording to be fixed forever, but the message should clearly communicate that `{{outlet}}` is only valid in routing contexts.

### Ember ecosystem implications

#### Lint rules

`ember-template-lint` could add a rule for the component-template case, but this RFC does not require a lint rule.

The runtime assertion is still needed because linting is optional and because engine-related cases depend on runtime ownership and routability.

#### Ember Inspector and debuggability

This change should improve debuggability by making invalid outlet usage fail closer to the source of the problem.

No Ember Inspector changes are required.

#### Server-side rendering

The same assertion should apply in SSR environments. Invalid `{{outlet}}` usage should not behave differently between browser rendering and server rendering.

#### Ember Engines

This RFC distinguishes between routable and non-routable engines.

`{{outlet}}` should remain valid in routable engine templates where route templates can render into an outlet.

`{{outlet}}` should assert in a non-routable engine mounted with `{{mount}}`, because that engine does not have a router and cannot provide routing outlet behavior.

#### Addon ecosystem

Some addons may contain templates with invalid `{{outlet}}` usage. Those addons would need to migrate away from this pattern.

The migration is expected to be straightforward: remove the invalid `{{outlet}}` usage and use route templates for routing composition, or use normal component composition patterns for component-level layout.

#### IDE support

No new syntax or editor support is required.

## How we teach this

The main teaching point is that `{{outlet}}` is a routing primitive, not a general component composition primitive.

The Ember Guides should continue to introduce `{{outlet}}` in the routing section. They should make clear that:

- `{{outlet}}` belongs in route templates and routable engine templates.
- Components should not use `{{outlet}}`.
- Components should use normal component composition patterns instead, such as arguments, yielded blocks, named blocks, or contextual components.
- Non-routable engines mounted with `{{mount}}` should not use `{{outlet}}`, because they do not have a router.

This change should be presented as a clarification of existing Ember routing behavior rather than as a new feature.

For existing Ember users, the migration message should explain that if they see this assertion, they should move `{{outlet}}` into an appropriate route template, or replace it with component composition.

For example, component-level layout should use `{{yield}}` instead of `{{outlet}}`.

Before:

```gjs
// app/components/page-layout.gjs

<template>
  <header>
    ...
  </header>

  {{outlet}}
</template>
```

After:

```gjs
// app/templates/application.gjs

<template>
  <PageLayout>
    {{outlet}}
  </PageLayout>
</template>
```

```gjs
// app/components/page-layout.gjs

<template>
  <header>
    ...
  </header>

  {{yield}}
</template>
```

This keeps routing behavior in the route template and keeps the component responsible for layout.

## Drawbacks

This change may affect applications or addons that currently contain invalid `{{outlet}}` usage.

Those usages are unsupported, but making them assert can still create upgrade work for maintainers. Apps that currently have `{{outlet}}` in component templates or in non-routable engine templates would need to move that routing behavior into route templates, or replace it with component composition.

Because this changes invalid usage from allowed-at-runtime to an assertion, the rollout path should be considered. A deprecation-first approach may be appropriate if the framework team wants to give applications and addons time to migrate before this becomes a hard assertion.

## Alternatives

### Do nothing

Ember could continue allowing invalid `{{outlet}}` usage.

This avoids changing behavior, but it keeps the current confusing behavior. Users can continue to write invalid templates without a clear explanation of what is wrong.

### Add only a lint rule

A lint rule could catch some cases of invalid `{{outlet}}` usage, especially in component templates.

However, linting is optional and cannot reliably cover the non-routable engine case. A lint rule would be useful, but it should not be the only mechanism.

### Deprecate first

Ember could first introduce a deprecation warning for invalid `{{outlet}}` usage, then convert it into an assertion in a later major version.

This would give apps and addons time to migrate. It may be the best rollout path if the framework team considers this a behavior change.

## Unresolved questions

- Should this be introduced through a deprecation-first path, or as a direct assertion?
- Should this be framed as Ember 8 cleanup work?