---
stage: accepted
start-date: 2025-06-15T00:00:00.000Z # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: 
  - framework
  - typescript
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
suite: 
---

<!--- 
Directions for above: 

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
suite: Leave as is
-->
# Service Helper Manager

## Summary

Add helper manager capabilities to `@ember/service`'s `service` export, enabling direct template usage while maintaining backward compatibility with existing service injection patterns.

## Motivation

Current service access in templates requires unnecessary boilerplate:

```js
import Component from '@glimmer/component';
import { service } from '@ember/service';

export default class extends Component {
  @service theme;
  
  // Boilerplate just to expose the service
  get exposedTheme() {
    return this.theme;
  }
}
```

```hbs
{{this.exposedTheme.toggle}}
```

With helper manager capabilities, eliminate the boilerplate:

```gjs
import Component from '@glimmer/component';
import { service } from '@ember/service';

<template>
  {{#let (service "theme") as |theme|}}
    <button {{on "click" theme.toggle}}>Toggle Theme</button>
  {{/let}}
</template>

export default class extends Component {}
```

## Detailed design

### Enhanced `service` Export

The `service` export gains helper manager capabilities while preserving existing patterns:

```js
// All existing usage unchanged
@service theme;
@service('user-preferences') prefs;
theme = service('theme');
```

```gjs
// New template helper usage
<template>
  {{! Direct service access with let }}
  {{#let (service "theme") as |theme|}}
    <button {{on "click" theme.toggle}}>Toggle Theme</button>
  {{/let}}
  
  {{! Dynamic service names }}
  {{#let (service @serviceName) as |dynamicService|}}
    {{dynamicService.someMethod}}
  {{/let}}
  
  {{! Composition with iteration }}
  {{#let (service "cart") as |cart|}}
    {{#each cart.items as |item|}}
      <div>{{item.name}} - {{cart.formatPrice item.price}}</div>
    {{/each}}
  {{/let}}
  
  {{! Property access }}
  <div class={{if (service "theme").isDark "dark" "light"}}>
    Content here
  </div>
</template>
```

### Implementation

```js
class ServiceHelperManager {
  createHelper(state, { positional: [serviceName] }) {
    return { service: getOwner(state).lookup(`service:${serviceName}`) };
  }
  
  getValue({ service }) {
    return service;
  }
}
```

### Template-only Components

Perfect for template-only components:

```gjs
<template>
  {{#let (service "cart") as |cart|}}
    <span class="cart-count">{{cart.totalItems}}</span>
    <span class="cart-total">{{cart.formattedTotal}}</span>
  {{/let}}
  {{yield}}
</template>
```

### Common Patterns

```gjs
<template>
  {{! Service method calls require let }}
  {{#let (service "formatter") as |formatter|}}
    <span>{{formatter.currency @price}}</span>
    <time>{{formatter.date @timestamp}}</time>
  {{/let}}
  
  {{! Multiple services }}
  {{#let (service "cart") (service "theme") as |cart theme|}}
    <div class={{theme.currentTheme}}>
      <p>Items: {{cart.totalItems}}</p>
      <button {{on "click" cart.clear}}>Clear Cart</button>
    </div>
  {{/let}}
</template>
```

### TypeScript Support

Full type inference:

```gts
<template>
  {{#let (service "theme") as |theme|}}
    {{! TypeScript knows theme is ThemeService }}
    <button {{on "click" theme.toggle}}>{{theme.buttonText}}</button>
  {{/let}}
</template>
```

## How we teach this

### Mental Model

"Service gets you a service" - consistent whether in classes or templates. Template usage follows standard Handlebars patterns with `{{#let}}` for method calls.

### Documentation

Update `@ember/service` docs with template examples. Add Guides section on "Services in Templates" emphasizing the `{{#let}}` pattern for method calls.

## Drawbacks

- **API Surface**: Another way to access services
- **Template Verbosity**: `{{#let}}` adds nesting for method calls

## Alternatives

Keep requiring backing class injection. Maintains separation but creates boilerplate.

This leads to utilities like [service in reactiveweb](https://reactive.nullvoxpopuli.com/functions/resource_service.service.html)

## Unresolved questions

- Error handling for missing services?
  - Would be more doable if we had try/catch in templates 
- Build-time vs runtime service validation?
  - forbid dynamic service names? 
