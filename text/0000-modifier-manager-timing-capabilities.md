---
stage: accepted
start-date: 
release-date: 
release-versions:
teams: # delete teams that aren't relevant
  - framework
  - learning
prs:
  accepted: # update this to the PR that you propose your RFC in
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

# Modifier Manager timing capability

## Summary

Element modifiers enable developers to customize a HTML element. Currently modifiers are always installed _after_ the browser painted the element. This could lead to a flash of unstyled content and negatively impact performance due to double paint. This RFC proposes adding a second timing option allowing modifiers to run _before_ the browser painted the element.

## Motivation

Delaying modifier install hook execution until browser painted the element, prevents modifiers from blocking rendering. Adding work into critical rendering path needs to be done carefully. If the browser is not able to execute all work within one rendering frame, the browser starts dropping frames, which could cause stuttering animations and slow feedback to user interactions. Delaying modifier Installation is an important performance optimization for most modifiers. But it may cause issues for modifiers, which either affect the layout of an element or add interactivity to it.

A modifier may be used to style an element. Painting the element _before_ the modifier is installed causes a flash of unstyled content. If this could be noticed by the user depends on many factors - including the performance of the device and other work executed in the same rendering frame.

Additionally, it causes a double paint issue. The browser needs to paint the same element twice: Once before and once after the modifier is installed. This contradicts the performance considerations, which leads to the timing decision on the first place.

Modifiers, which affect the layout of an element, are common. Example use cases, which have been published as open source, includes:

- The [ember-style-modifier](https://emberobserver.com/addons/ember-style-modifier), which allows to modify the CSS styles of an element. It is one of the most used modifiers accordingly to NPM download stats.
- The [ember-autoresize-modifier](https://emberobserver.com/addons/ember-autoresize-modifier), which resizes a textarea to fit its content.
- ...

Installing a modifier after browser painted the element also causes issues for modifiers, which adds interactivity to an element. The `{{on}}` modifier provided by Ember itself is an example of such a modifier. It registers event listeners on an element. If an user interact with an element _before_ the event listeners are registered may lead to a broken user experience.

This RFC aims to overcome this limitation by allowing modifiers to request execution _before_ browser painted the element. This timing will be called _on layout_ going forward. The existing timing will be called _on idle_.

## Detailed design

This RFC proposes adding a new modifier capability. The new capability adds two new hooks to modifier managers:

1. `installModifierOnLayout` 
2. `installModifierOnIdle`

Both hooks receive the same arguments as the existing `installModifier` hook. They only differ in their timing.

The existing `installModifier` does not exist anymore in that new capability.

### `installModifierOnLayout`

The `installModifierOnLayout` has the following timing:

**Always**

- called after all children modifier managers `installModifer` hook are called
- called after DOM insertion
- called in the same tick as DOM insertion 

**May or May Not**

- have the sibling nodes fully initialized in DOM
 
### `installModifierOnIdle`

The `installModifierOnIdle` has the following timing:

**Always**

- called after all childrens modifier managers `installModifierOnLayout` hook are called

**May or May Not**

- be called in the same tick as DOM insertion
- have the sibling nodes fully initialized in DOM

This is de facto the same timing as the existing `installModifier` hook has. Existing modifiers can upgrade to `installModifierOnIdle` without seeing any change in their functionality.

### `updateModifier`

The existing `updateModifier` hook stay as is. There is no need for different timing for updates. The element is already rendered. A double paint issue cannot occur.

## How we teach this

Modifier managers are a low-level primitive. We don't expect application developers to use them directly. The API is meant to be used by libraries such as [ember-modifiers](https://github.com/ember-modifier/ember-modifier), which provides APIs with great developer experience for them. Modifier managers are not covered on the guides for that reason.

`ModifierManager` should be covered in API documentation. But it is [currently not](https://github.com/emberjs/ember.js/issues/20273). When missing API documentation is added, it must reflect the changes to the public API introduced by this RFC.

## Drawbacks

### Not addressed timing requests

The need of two additional timing capabilities were raised in [preparation of this RFC](https://github.com/emberjs/rfcs/issues/652#issuecomment-1195772115):

1. Running modifiers _before_ the element is insert into the DOM. Reliable setting properties on custom elements ("web components") was raised as an use case for such a timing.  Custom elements have [`connectedCallback` lifecycle hook](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_custom_elements#using_the_lifecycle_callbacks), which is executed when the element is insert into the DOM. A custom element developed by a third-party may rely on a property being set when that lifecycle hook is executed.
2. Running modifiers in order of template invocation. Grabbing a reference to an element to be used with [`{{in-element}}`](https://api.emberjs.com/ember/release/classes/Ember.Templates.helpers/methods/each?anchor=in-element) helper or another modifier was raised as the main use case.

These RFC does not propose adding those timings. In opposite to the _on layout_ timing, running modifiers _before_ the element is insert into the DOM or even in order of template may introduce performance issues. Additionally it is not clear yet if server-side rendering should be supported for those and how such a support could look like. We may want to address these use cases with different concepts. Addressing those use cases is left for follow-up RFCs.

### Others

Adding two new hooks and removing one requires application and addons to migrate code. This is limited as applications typically do not implement modifier managers themselves. Instead most addons and application implement modifiers using high-level APIs provided by [ember-modifier](https://github.com/ember-modifier/ember-modifier). This encapsulates most of the upgrade costs at one single library.

Most of the existing modifiers are implemented using `ember-modifier`. Modifiers would either need to be refactored to implement modifier manager directly or would need to wait until `ember-modifier` supports them.

Introducing two different timing for modifiers, on layout and on idle, requires the developer to reason about which one fits the use case best. This could be seen as increasing complexity and mental load on developers.

## Alternatives

1. We could continue accepting the unnecessary double paint and it's potential negative impact on performance.
2. We could change the timing of the existing `installModifier` hook to run on layout always. Doing so would resolve the double paint issue. But we would accept that some modifiers are executed in critical rendering path even if not needed.

## Unresolved questions

No one uncovered so far.
