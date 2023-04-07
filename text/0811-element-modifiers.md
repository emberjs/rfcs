---
stage: ready-for-release
start-date: 2022-03-29T00:00:00.000Z
release-date:
release-versions:
teams:
  - cli
  - learning
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/811'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/885'
project-link:
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
-->

# Element Modifiers

## Summary

This RFC introduces the concept of user defined element modifiers and proposes
adding [ember-modifier](https://github.com/ember-modifier/ember-modifier)
to the blueprint that back `ember new`, providing an
officially-supported path for using modifiers out of the box.

This RFC supersedes the original [RFC #353 "Modifiers"][rfc-0353].

## Motivation

Ember Octane introduced Glimmer Components as a replacement for Classic
Components. They are simpler, more ergonomic, and more declarative. In contrast
with Classic Components, Glimmer Components don't have any element/DOM based
properties or hooks giving access to DOM. This was an intentional move as
component backing class gets disconnected from DOM manipulation.

Modifiers are similar to template helpers: they are functions or classes that can be used
in templates directly using `{{double-curlies}}` syntax. The major difference with modifiers
is that they are applied directly to *elements*:

```handlebars
<button {{effect "fade-in"}}>Save</button>
```

This RFC builds on top of low-level primitives for defining element modifiers
introduced in [RFC #373 "Element Modifier Manager"][rfc-0373].
The `ember-modifier` addon was built on top of those low-level primitives and
provides a convenient API for authoring element modifiers in Ember.

Element Modifiers were introduced as part of Ember Octane edition and are first class citizens
in Ember application, like components or helpers. However today, developers need to install
an additional library to be able to define custom modifiers.

This RFC seeks to fill this gap in Ember.js' development mental model by
providing `ember-modifier` in the blueprint. `ember-modifier` will be added
to `devDependencies`, same as e.g. @glimmer/component.

A basic element modifier is defined with the type of `modifier`.
For example these paths would be global element modifiers in an application:

* `app/modifiers/draggable.js`
* `app/modifiers/effect.js`

Similar to helpers, modifiers can be function-based:

```js
// /app/modifiers/on.js
import { modifier } from 'ember-modifier';

export default modifier((element, [eventName, handler]) => {
  element.addEventListener(eventName, handler);

  return () => {
    element.removeEventListener(eventName, handler);
  }
});
```

or class-based

```ts
// /app/modifiers/on.js
import Modifier from 'ember-modifier';
import { registerDestructor } from '@ember/destroyable';

function cleanup(instance: OnModifier) {
  let { element, event, handler } = instance;

  if (element && event && handler) {
    element.removeEventListener(event, handler);

    instance.element = null;
    instance.event = null;
    instance.handler = null;
  }
}

export default class OnModifier extends Modifier {
  element = null;
  event = null;
  handler = null;

  // Manage teardown
  constructor() {
    super(...arguments);
    registerDestructor(this, cleanup);
  }

  // Run on installation and all changes to tracked state.
  modify(element, [event, handler]) {
    // Clear any previous state.
    cleanup(this);

    // Set up the new handling.
    this.addEventListener(element, event, handler);
  }

  // methods for reuse
  addEventListener(element, event, handler) {
    // Store the current element, event, and handler for when we need to remove
    // them during cleanup.
    this.element = element;
    this.event = event;
    this.handler = handler;

    element.addEventListener(event, handler);
  };
}
```

While this is slightly more complicated than the function-based version,
that complexity comes along with much more *control*. It also allows us to hook
into Ember's dependency injection system, for example to use services.

[RFC #432: Contextual Helpers and Modifiers][rfc-0432] allows optional
invocation and partial application of modifiers, and allows modifiers to be
yielded, passed as arguments, etc. [RFC #779: First-Class Component
Templates][rfc-0779] allows defining them locally or importing them directly to
be used in a `<template>` context. This combination makes many patterns much
easier to implement:

```js
// /app/components/slide-up-card.js
import Component from '@glimmer/component';
import { modifier } from 'ember-modifier';

const lockBodyScroll = modifier((element, [shouldLockBodyScroll]) => {
  document.body.classList.add('lock-scroll');
  return () => document.body.classList.remove('lock-scroll');
});

export default class SlideUpCard extends Component {
  @action closeCard (event) {
    event.preventDefault();
    this.args.close();
  }

  <template>
    {{#if has-block}}
      {{yield (hash lock=lockBodyScroll onClick=this.closeCard)}}
    {{else}}
      <div
        class="slide-up-card"
        ...attributes
        {{lockBodyScroll}}
      >
        <p>{{@bodyContent}}</p>

        <button {{on "click" this.closeCard}}>Dismiss</button>
      </div>
    {{/if}}
  </template>
}
```

## Detailed design

The necessary changes to `ember-cli` are relatively small since we only need
to add the dependency to the `app` blueprint.

Note that `addon` blueprint will not include `ember-modifier` due to
unresolved question (at the time of writing this RFC) regarding how addons
should declare dependencies like `@glimmer/component`, `@glimmer/tracking`, `ember-modifier` etc.

This has the advantage (over including it as an implicit dependency), that
apps that don't want to use it for some reason can opt out by
removing the dependency from their `package.json` file.

**Notes:**
1. This is *not* the usual path for delivering features into Ember: we do not
   generally introduce community addons directly into the blueprint. In this
   specific case, we think it is the right move:

    - The addon has intentionally been reworked to rationalize it in
      terms of the rest of the Octane programming model, explicitly with an eye to adoption in this way.
    - We do *not* want to introduce `@glimmer/modifier` with exactly this
      API, at least not yet, because we believe we may want to introduce
      a slightly updated API from that package *without* requiring
      breaking changes in the future.
2. We are intentionally introducing more "incoherence" to the programming
   model with this addition. (Or rather, we are acknowledging the *existing*
   incoherence in the ecosystem, with an eye to making progress on resolving
   it!) In particular, as the community begins adopting strict mode templates
   via First-Class Component Templates in the months ahead, they will often
   end up importing from both `ember-modifier` and `@ember/modifier`:

   ```js
   import { on } from '@ember/modifier';
   import { modifier } from 'ember-modifier';
   const playWhen = modifier((el, [shouldPlay]) => {
     if (shouldPlay) {
       el.play();
     } else {
       el.pause();
     }
   });
   <template>
     <audio
       src={{@src}}
       {{on "error" @onBadLoad}}
       {{playWhen @shouldPlay}}
     />
   </template>
   ```

   As noted in (1) above, we have ideas on how to resolve this, but are
   intentionally not blocking on those in favor of unlocking this key
   functionality for Ember users in the Octane programming model *today*.

## How we teach this

Ember Guides already teach how to use modifiers in "Template Lifecycle, DOM, and Modifiers"
section of the guides.

However, there are a couple changes we need to make to the content of the
guides as they stand today:

1. We will remove the "install `ember-modifier`" instruction from the guides,
   since it will already be part of the blueprint.
2. We should include at least a minimal example of a class-based modifier. For
   that example, we can use something like `setInterval` to show how setup,
   updates, and teardown can be more convenient when using a class.
   (Note that here we should *only* teach the [newly-redesigned API][v3.2.0], not
   the deprecated previous APIs. This will simplify our teaching significantly,
   since we can describe how it matches the `Helper` APIs.)

## Drawbacks

- If we merge [RFC #757 "Default Modifier Manager][rfc-0757],
  it may seem redundant with this RFC.
- There is the potential for confusion between the `ember-modifier` package and
  the `@ember/modifier` package which ships as part of `ember-source`.

## Alternatives

- Introduce [RFC #757 "Default Modifier Manager][rfc-0757]
  creating modifiers and ask developers to install `ember-modifier`
  for more complex use cases.
- Integrate `ember-modifier` into the `@ember/modifier` package directly.
- Introduce a new `@ember/*` or `@glimmer/*` package for the contents of
  `ember-modifier`.

## Unresolved questions

None.

[rfc-0353]: https://github.com/emberjs/rfcs/pull/353
[rfc-0373]: https://emberjs.github.io/rfcs/0373-Element-Modifier-Managers.html
[rfc-0432]: https://emberjs.github.io/rfcs/0432-contextual-helpers.html
[rfc-0757]: https://github.com/emberjs/rfcs/pull/757
[rfc-0779]: https://github.com/emberjs/rfcs/pull/779
[v3.2.0]: https://github.com/ember-modifier/ember-modifier/releases/tag/v3.2.0