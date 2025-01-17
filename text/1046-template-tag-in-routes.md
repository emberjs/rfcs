---
stage: released
start-date: 2024-10-04T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
  - learning
  - typescript
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1046'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/1053'
  released: 'https://github.com/emberjs/rfcs/pull/1069'
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

# Use Template Tag in Routes

## Summary

Allow `app/templates/*.hbs` to convert to `app/temlates/*.gjs`.

## Motivation

We are rapidly approaching the point where Template Tag is the recommended way to author components. This means using `.gjs` (or `.gts`) files that combine your template and Javascript into one file. But you cannot currently use Template Tag to author the top-level templates invoked by the router (`app/templates/*.hbs`).

This inconsistency is especially apparent when working on teaching materials for new users. Making people learn both `.hbs` with global component resolution and `.gjs` with strict template resolution before they can even make their first component is unreasonable.

This RFC proposes allowing consistent use of `.gjs` everywhere. It doesn't remove any support for `.hbs`, but recommends that the guides default to all `.gjs`.

## Detailed design

The [implementation is small and already done](https://github.com/emberjs/ember.js/pull/20768).

### Illustration By Example

If you currently have this:

```hbs
{{! app/templates/example.hbs }}
<article>
  <MainContent @model={{@model}} @editorMode={{this.editorMode}} />
</article>
```

You can convert it to this:

```gjs
// app/templates/example.gjs
import MainContent from 'my-app/components/main-content';
<template>
  <article>
    <MainContent @model={{@model}} @editorModel={{@controller.editorMode}} />
  </article>
</template>
```

Key differences:

- this is [strict handlebars](https://github.com/emberjs/rfcs/blob/master/text/0496-handlebars-strict-mode.md), so components are imported explicitly
- the controller is no longer `this`, it is `@controller`.

Many things that you might have been forced to put on a controller can now be done directly. For example, if your controller has a `doSomething` event handler:

```hbs
{{! app/templates/example.hbs }}
<div {{on "click" this.doSomething}}></div>
```

You now have options to implement it in-place in the same file. If it's stateless it can just be a function:

```gjs
// app/templates/example.gjs

// This import will be unnecessary after https://github.com/emberjs/rfcs/pull/1033
import { on } from '@ember/modifier';

function doSomething() {
  alert("It worked");
}

<template>
  <div {{on "click" doSomething}}></div>
</template>
```

If it's stateful, you can upgrade from a template-only component to a Glimmer component:

```gjs
// app/templates/example.gjs
import { on } from '@ember/modifier';
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';

export default class extends Component {
  @tracked activated = false;

  doSomething = () => {
    this.activated = !this.activated;
  }

  <template>
    <div {{on "click" this.doSomething }}></div>
    {{#if this.activated}}
      It's activated!
    {{/if}}
  </template>
}
```

### Specification

When Ember resolves a route template (`owner.lookup("template:example")`:

1. Check whether the resulting value has a component manager.
   - If no, do exactly what happens today.
   - If yes, continue to step 2.
2. Synthesize a route template that invokes your component with these arguments:
   - @model: which means exactly the same as always.
   - @controller: makes the controller available.

Keen observers will notice that this says nothing about only supporting gjs files. Any component is eligible, no matter how it's authored or what Component Manager it uses. This is by design, because there's no reason for the router to violate the component abstraction and care about how that component was implemented.

However, in our learning materials we should present this as a feature designed for GJS files. Using it with components authored in .hbs would be needlessly confusing, because automatic template co-location **does not work in app/templates**, because it would collide with traditional route templates.

### ember-route-template addon

This RFC replaces the [ember-route-template](https://github.com/discourse/ember-route-template) addon. If you're already using it, it would continue to work without breaking, but you can simply delete all the calls to its `RouteTemplate` function and remove it. The popularity of that addon among teams who are already adopting Template Tag is an argument in favor of this RFC.

### Codemod

Because Embroider already generates imports for components, helpers, and modifiers in non-strict templates, there is [ongoing work](https://github.com/embroider-build/embroider/pull/2134) to offer Embroider's existing functionality as a Template Tag codemod.

For route templates, the only extra feature required would be replacing `this` with `@controller`.

In order to help people be successful with the codemod, we should also:

- deprecate passing dynamic component strings to the `{{component ...}}` helper, since that is the only non-strict-handlebars feature that Embroider cannot automatically concert to the equivalent strict-handlebars code.

- continue to emphasize v2 addons, because v1 addons that use the full weird panoply of old behaviors can make the codemod unreliable.

### TypeScript

No new typescript-specific features are required. If you're authoring route templates in GTS, Glint should treat them just like any other component. You will need to manually declare the signature for `@model` and `@controller`, but that is the same as now.

## How we teach this

This RFC was directly inspired by a first attempt at updating the Guides for Template Tag. It became immediately apparent that we can write _much_ clearer guides if we can teach all GJS, instead of a mix of GJS and HBS.

Starting at https://guides.emberjs.com/release/components/ when the first `application.hbs` file is introduced, we would use `.gjs` instead. In those opening examples that are empahsizing HTML, the only change to the content would be wrapping `<template></template>` around the HTML.

Progressing to https://guides.emberjs.com/release/components/introducing-components/, learners extract their first component. It now becomes possible to do that within the same file. This allows teaching fewer things in a single step. First, people can learn what a component is. Second, it can be refactored into a separate file. We can avoid teaching anything about "pascal case" and naming rules, because everything just follows javascript naming rules.

When starting to teach routing in https://guides.emberjs.com/release/routing/defining-your-routes/, the file extensions change and `<template></template>` wrappers are added, but nothing else on that page necessarily changes.

In https://guides.emberjs.com/release/routing/query-params/, it's appropriate to first introduce the `@controller` argument.

In https://guides.emberjs.com/release/routing/controllers/, the list of reasons to use a controller gets shortened to only queryParams, since now you can manage state directly in your route's component.

### How to teach: what to do when you encounter an HBS route template?

Guides will need one or more callout boxes in the routing area to point people toward a dedicated page about HBS files in app/templates.

The dedicated page will explain that this is the older pattern, the controller is available as `this` the model is still `@model`, and the instruction for dealing with them is to run the codemod to convert them to GJS.

## Drawbacks

There is appetite for a more ambitious RFC that changes more things about routing. Eliminating controllers, making routes play nice with the newer `@ember/destroyable` system, allowing parallel model hooks, etc, are all good goals. There is a risk that if we do those things soon, this would be seen as two steps of churn instead of one.

I think we can mitigate that risk because

- we won't deprecate `.hbs` routes yet, so no churn is forced immediately.
- we can ship the Template Tag codemod so that even big apps can adopt at low cost
- it's extremely unlikely that a future routing design would use anything other than `.gjs` to define route entrypoints. By converting now, you are already moving in the right direction by eliminating all the non-strict behaviors.

## Alternatives

### Bigger Router RFC

The main alternative here is to do a bigger change to the routing system. A "Route Manager" RFC would allow the creation of new Route implementations that could have their own opinions about how to route to GJS files. This RFC does not preclude that other work from also happening.

The main benefit of this RFC is that the [implementation is small and already done](https://github.com/emberjs/ember.js/pull/20768) so we could have it immediately.

### Eliminate Bare Templates Entirely

The existence of "bare templates" in the system alongside HBS components is a major source of incoherence and potential confusion. It's especially bad that the behavior is hard-coded to depend on particular filesystem paths (a standalone hbs file in `app/components` gets built to a component whereas anywhere else it remains a bare template).

To eliminate this source of incoherence, it would be desirable to introduce a feature flag that would make _all_ hbs files interpreted as components, regardless of filesystem path.

This is probably worth doing regardless of whether we also do the present RFC and deserves its own separate proposal. There is a practical question of whether this will be faster than eliminating all HBS via codemod and deprecation and removal.
