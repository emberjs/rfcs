---
Stage: Accepted
Start Date: 03/13/2021
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: #726
---

<!--- 
Directions for above: 

Stage: Leave as is
Start Date: Fill in with today's date, YYYY-MM-DD
Release Date: Leave as is
Release Versions: Leave as is
Relevant Team(s): Fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies
RFC PR: Fill this in with the URL for the Proposal RFC PR
-->

# DOM Element descriptor interface for test helpers

## Summary

[@ember/test-helpers](https://github.com/emberjs/ember-test-helpers)' DOM helpers and [qunit-dom](https://github.com/simplabs/qunit-dom) accept either a selector or an `Element` as their DOM element descriptor, which is a pretty common pattern. This RFC proposes an interface that generalizes the notion of a DOM element descriptor that `@ember/test-helpers`' DOM helpers, `qunit-dom`, and other similar test support libraries in the Ember ecosystem can support, so page object implementations such as [fractal-page-object](https://github.com/bendemboski/fractal-page-object) and [ember-cli-page-object](http://ember-cli-page-object.js.org/) can implement the interface and integrate cleanly and ergonomically with these test support libraries.

## Motivation

In this RFC we'll use **DOM element descriptor** to mean a Javascript value that can be used to retrieve one or more `Element`s from a rendered DOM. Examples include a string containing a CSS selector, a direct reference to an `Element`, a function that returns an `Element` or `null`, a function that returns an array of `Element`s, etc.

In this RFC we'll use **DOM helper** to mean an API that is passed a DOM element descriptor, and performs some actions based on the element or elements the descriptor describes. Examples include `@ember/test-helpers`' DOM helper methods such as `click()` and `triggerEvent`, and `qunit-dom`'s assertion factory API, i.e. `assert.dom()`.

Ember's testing ecosystem has converged on a single set of recommended DOM helpers. `@ember/test-helpers` provides the core value of implementing DOM interactions, while `qunit-dom` provides the core value of making assertions about DOM query results. They do _not_ (nor should they) provide the core value of advanced DOM querying -- they act on DOM elements, and do the simple baseline of supporting two kinds of DOM element descriptors, CSS selector strings and `Element` references.

[Page Objects](https://martinfowler.com/bliki/PageObject.html) are a powerful and popular tool for making test code well organized, readable, and maintainable. The core value they provide is supporting the implementation of a domain-specific languange (DSL) for interacting with a rendered DOM. Such DSLs can be thought of as, among other things, providing powerful and resuable DOM element descriptors.

Supporting a clean integration of page objects (or, indeed, anything that produces DOM element descriptors that aren't CSS selectors or direct `Element` references) with DOM helpers like the ones in `@ember/test-helpers` and `qunit-dom` would keep each focused on its core value, while providing "first-class" support for page objects and other DOM element descriptors.

As a case study (and most proximate motivation for this RFC), let's consider `fractal-page-object`. Currently `fractal-page-object` integrates with `@ember/test-helpers` and `qunit-dom` by exposing an `element` property on all page objects, and that works because their APIs accept direct `Element` references. But this has three major drawbacks:

1. It only supports single elements. While this isn't an issue with the `@ember/test-helpers` DOM APIs, in `qunit-dom` that means there's no way to use the multi-element assertions, analogous to  `assert.dom(document.querySelectorAll('.list-item')).exists({ count: 3 })`.
2. It does not produce useful information when encountering an error/failure. For example, `await click('.does-not-exist')` errors with `Error: Element not found when calling click('.does-not-exist').`, while `await click(document.querySelector('.does-not-exist'))` errors with the less helpful `Error: Must pass an element or selector to 'click'` (because it doesn't have any other information to show -- all it knows is it was passed `null` instead of a selector or `Element`). Similarly, `assert.dom('.does-not-exist').exists()` produces the assertion message `Element .does-not-exist exists`, while `assert.dom(document.querySelector('.does-not-exist')).exists()` produces the less helpful `Element <not found> exists`.
3. It supports a noticeably less ergonomic integration with page objects. In `fractal-page-object`, the integrations currently look like `await click(page.listItems[2].checkbox.element)` and `assert.dom(page.listItems[2].checkbox.element).isChecked()`, rather than `await click(page.listItems[2].checkbox)` and `assert.dom(page.listItems[2].checkbox).isChecked()`. While it may look like a minor difference, in practice it's quite a stumbling block as the mental model of page objects encourages users to think of page object nodes (e.g. `page.listItems[2].checkbox`) as a proxy for the DOM element(s) they describe, so forgetting to add the `.element` is extremely common, even for the author of the library!

We can solve these by formalizing the notion of DOM element descriptors in an interface, so page object implementations and any other mechanisms for DOM access can produce objects implementing this interface and they can be passed to these test helpers without incurring any of the shortcoming described above.

## Detailed design

### New use cases

We're seeking to address two specific new use cases, although the design should be extensible to other unforeseen use cases as well.

#### Page objects

Page objects can be used as DOM element descriptors and passed directly to DOM helpers:

```js
assert.dom(pageObject.listItems).exists({ count: 4 });

await click(pageObject.listItems[2].checkbox);

assert.dom(pageObject.listItems[2].checkbox).isChecked();
```

This will enable more ergonomic integrations of libraries like `ember-cli-page-object` and `fractal-page-object` with DOM helpers.

#### Ad-hoc descriptors

DOM element descriptors can be constructed directly from `Element`s:

```ts
let element = someOtherLibrary.getGraphElement();
let descriptor = createDOMDescriptor({ element, description: 'graph element' });

await click(descriptor);

assert.dom(descriptor).hasClass('selected');
```

or from simple classes for lazy DOM lookups:

```ts
class MyDescriptor {
  get element() {
    return document.querySelectorAll('.list-item')[2];
  }
  description = 'second list item';
}
let descriptor = new MyDescriptor();

assert.dom(descriptor).doesNotExist();

await click('.add-list-item');

assert.dom(descriptor).exists();
```

This will enable more flexible queries than CSS selectors support without losing descriptive debug/assertion messages.

### Namespacing constraints

The page object use case introduces a significant constraint. Page objects are necessarily extensible, allowing users to define their own properties on them, so a DOM element descriptor interface that relies on properties stored on the object implementing the interface risks conflicts with user-defined properties. For example, if a user implements an album info page object for a page that shows information on an album, it might have a `description` property exposing the rendered text of the album's description, which would mean that it could not also implement an interface that requires it to expose the DOM element descriptor's description under a `description` property.

To address this, we conceive of DOM element descriptors as arbitrary objects whose descriptor data is registered and stored in some private fashion that avoids namespace concerns (e.g. in a `WeakMap`).

### The design

This RFC proposes implementing a library that exports two core functions, and several other convenience methods to improve ergonomics.

#### Core functions

```ts
interface IDOMElementDescriptor {}

interface DescriptorData {
  element?: Element | null;
  elements?: Element[];
  description?: string;
}

function registerDescriptorData(descriptor: IDOMElementDescriptor, data: DescriptorData): void;
function lookupDescriptorData(descriptor: IDOMElementDescriptor): DescriptorData;
```

(the interfaces have been simplified a bit for illustrative purposes -- in particular, `DescriptorData` would need to enforce that at least one of `element` and `elements` is defined)

`IDOMElementDescriptor` is a "no-op interface" -- it has no properties or methods, and only exists to support typing. So typescript concerns aside, it can be thought of as just `object`.

`DescriptorData` is a type that contains an `element` property and/or an `elements` property, and also an optional `description` property. The `element` and `elements` properties exist to support usage in both single-element contexts (the equivalent of passing a selector to `querySelector()`) and multi-element contexts (the equivalent of passing a selector to `querySelectorAll()`). At least one of them must be defined, and both may be defined. If only the `element` property is defined, then multi-element contexts should act as if the `elements` property were defined to be either an empty array or a singleton array, depending on whether the `element` property evaluates to `null` or an `Element`. If only the `elements` property is defined, then single-element contexts should act as if `element` were defined to be the first element of the `elements` property, or `null` if the `elements` property evaluates to an empty array. To illustrate further, here are two possible implementations of functions to resolve descriptors to DOM elements:

```ts
function getDescriptorElement(data: DescriptorData): Element | null {
  if (data.element !== undefined) {
    return data.element;
  } else {
    return data.elements[0] || null;
  }
}

function getDescriptorElements(data: DescriptorData): Element[] {
  if (data.elements) {
    return data.elements;
  } else {
    let element = data.element;
    return element ? [element] : [];
  }
}
```

It would be possible to only support an `elements` property and always have single-element contexts determine their element as described above, but since the vast majority of current DOM helpers are single-element, we allow `DescriptorData` instances to define both to allow optimizations, e.g. a class implementing `element` and `elements` as getters could call `querySelector()` instead of `querySelectorAll()` when `element` is accessed.

Producers of DOM element descriptors, like page objects or test code producing ad-hoc DOM element descriptors, will use `registerDescriptorData()` to associate data with descriptors, and DOM helpers that are passed descriptors will use `lookupDescriptorData()` to retrieve the data for a given descriptor.

#### Convenience methods

The library will include a convenience function for creating ad-hoc DOM element descriptors:

```ts
function createDescriptor(data: DescriptorData): IDOMElementDescriptor
```

which would create an `IDOMElementDescriptor`, use it to register the data, then return it. No equivalent is required for page objects, as the expectation is that they will themselves be `IDOMElementDescriptors`, so they will only need to perform the registration step from their constructor.

The library will also include some functions for use by DOM helpers when using DOM element descriptors to access the DOM. They can use `lookupDescriptorData()` directly, but as mentioned above, that would involve some "boilerplate" code for resolving the data to actual DOM elements in single- or multi-element contexts, since the descriptor data might only have one of `element` or `elements` defined. So the library will implement

```ts
function resolveDOMElement(target: IDOMElementDescriptor | DescriptorData): Element | null;
function resolveDOMElements(target: IDOMElementDescriptor | DescriptorData): Element[];
```

It may make sense to implement some kind of

```ts
function getDescription(target: IDOMElementDescriptor | DescriptorData): string;
```

function for returning the descriptor's `description` or deriving some kind of reasonable default description from the descriptors elements, but that's outside the scope of this RFC, and only mentioned here to help paint the broader picture. It may also make sense to implement additional types and helpers to streamline the boilerplate arguments-resolving logic in DOM helpers that accept `Element`s, CSS selector `string`s, and `IDOMElementDescriptors`.

## How we teach this

This is primarily taught through the API documentation of the various libraries that implement compliant DOM helpers, such as `@ember/test-helpers` and `qunit-dom`, and libraries that produce DOM element descriptors, such as `fractal-page-object`. In addition the new library proposed in this RFC would have documentation explaining the infrastructure and how to use it.

The Ember guides would not need to be updated, as the default Ember testing methodology would be unaffected -- this would be added functionality for users using page object libraries, or implementing ad-hoc element descriptors in their tests.

## Drawbacks

* This would increase the complexity of the DOM helper and assertion APIs
* This would add a small amount of extra maintenance cost to those libraries as all new code does

## Alternatives

### Do nothing

This mainly improves developer ergonomics by allowing for better error/failure messaging and a simpler/more natural semantics when passing page objects to DOM helper/assertion methods. We could decide that the status quo is good enough, and the improved ergonomics are not worth the cost.

### Symbol for data storage

Instead of using a private data storage to associate the descriptor data with the `IDOMElementDescriptor`s, we could export a `Symbol` to solve the namespacing problem:

```ts
export const DESCRIPTOR_DATA = Symbol('descriptor data');

export interface IDOMElementDescriptor {
  [DESCRIPTOR_DATA]: DescriptorData;
}
```

This is maybe a slightly more familiar pattern to some, and might make debugging slightly easier, but doesn't seem to confer any real benefits. Moreover, the internal implementation of `registerDescriptorData` and `lookupDescriptorData` could store the state on the `IDOMElementDescriptor` object using a private `Symbol` if that were discovered to be advantageous in the future.

### One interface, three symbols

We could get rid of the `DescriptorData` type entirely and modify the `IDOMElementDescriptor` interface to store `element`, `elements`, and `description` on properties keyed by three different public `Symbol`s. This doesn't seem to confer any real benefits that aren't captured by the primary proposal or the `Symbol for data storage` alternative, and makes ad-hoc descriptors less ergonomic:

```ts
let element = someOtherLibrary.getGraphElement();
let descriptor = {
  [ELEMENT]: element,
  [DESCRIPTION]: 'graph element'
};
await click(descriptor);
assert.dom(descriptor).hasClass('selected');
```

## Unresolved questions

* What do we call the new library?
