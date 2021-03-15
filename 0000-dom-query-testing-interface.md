---
Stage: Accepted
Start Date: 03/13/2021
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: 
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
3. It supports a much less ergonomic integration with page objects. In `fractal-page-object`, the integrations currently look like `await click(page.listItems[2].checkbox.element)` and `assert.dom(page.listItems[2].checkbox.element).isChecked()`, rather than `await click(page.listItems[2].checkbox)` and `assert.dom(page.listItems[2].checkbox).isChecked()`. While it may look like a minor difference, in practice it's quite a stumbling block as the mental model of page objects encourages users to think of page object nodes (e.g. `page.listItems[2].checkbox`) as a proxy for the DOM element(s) they describe, so forgetting to add the `.element` is extremely common, even for the author of the library!

We can solve these by formalizing the notion of DOM element descriptors in an interface, so page object implementations and any other mechanisms for DOM access can produce objects implementing this interface and they can be passed to these test helpers without incurring any of the shortcoming described above.

## Detailed design

### The interfaces

This RFC proposes implementing a very lightweight library that exports one `Symbol` and two interfaces:

```ts
export const ELEMENT_DESCRIPTOR = Symbol('element descriptor');

export interface IElementDescriptor {
  element: Element | null;
  elements: Element[];
  description?: string;
}

export interface IElementDescriptorProvider {
  [ELEMENT_DESCRIPTOR]: IElementDescriptor;
}
```

Then any API method that accepts a selector or element can also accept an `IElementDescriptor` or an `IElementDescriptorProvider`. For example a `click()` method like the one in `@ember/test-helpers` could be implemented as

```ts
function click(target: string | Element | IElementDescriptor | IElementDescriptorProvider) {
  if (!target) {
    throw new Error('Must pass an element or selector to `click`.');
  }
  
  let element;
  if (target instanceof Element) {
    element = target;
  } else if (typeof target === 'string') {
    element = document.querySelector(target);
    if (!element) {
      throw new Error(`element not found when calling \`click(${selector})\`.`);
    }
  } else {
    // IElementDescriptor or IElementDescriptorProvider
    let descriptor = target[ELEMENT_DESCRIPTOR];
    if (!descriptor) {
      descriptor = target;
    }
    
    let element = descriptor.element;
    if (!element) {
      let description = descriptor.description;
      if (description) {
        throw new Error(`element not found when calling \`click()\` with descriptor: ${description}`);
      } else {
        throw new Error(`element not found when calling \`click()\` with descriptor`);
      }
    }
  }

  element.click();
}
```

To further illustrate, this is an (unnecessary) implementation of a descriptor and descriptor factory that just wrap a selector:

```ts
class SelectorDescriptor implements IElementDescriptor {
  constructor(private selector: string) {}
  
  get element(): Element | null {
    return document.querySelector(this.selector);
  }
  
  get elements(): Element[] {
    return document.querySelectorAll(this.selector);
  }
  
  get description(): string {
    return this.selector;
  }
}

class SelectorDOMQuery implements IElementDescriptorProvider {
  public elementDescriptor: IElementDescriptor;

  constructor(selector: string) {
    this.elementDescriptor = new SelectorDescriptor(selector);
  }
}
```

This RFC proposes to extend `@ember/test-helpers`'s DOM helpers and `qunit-dom`'s DOM assertions to accept arguments of type `IElementDescriptor` and `IElementDescriptorProvider` anywhere they accept a selector or an `Element`.

### Why a symbol? Why two interfaces?

It would be possible to only have the `IElementDescriptor` interface and not the `IElementDescriptorProvider` interface, and do away with the `ELEMENT_DESCRIPTOR` symbol. The short answer for why we have them is that one of this RFC's target use cases is page objects providing element descriptors, and if they _were_ element descriptors then the element descriptor interface properties/methods would be in the same namespace as user-defined properties on the page objects (e.g. a user might want to implement a `description` property on an album page object containing the text of the album's description, and this would conflict with the `description` property on the `IElementDescriptor` interface). So the `IElementDescriptorProvider` interface and `ELEMENT_DESCRIPTOR` symbol exist to allow such page objects to avoid namespace conflicts.

This is further discussed in the alternatives section of this RFC.

### New use cases

Page objects:

```js
await click(pageObject.listItems[2].checkbox);
assert.dom(pageObject.listItems[2].checkbox).isChecked();
```

Ad-hoc descriptors:

```js
let element = someOtherLibrary.getGraphElement();
await click({ element, elements: element ? [element] : [], description: 'graph element' });
assert.dom({ element, elements: element ? [element] : [], description: 'graph element' }).hasClass('selected');
```

It would probably make sense to implement some factory functions for ad-hoc descriptors, e.g.

```ts
function descriptorFromElement(element: Element | null, description?: string): IElementDescriptor;
function descriptorFromElements(elements: Element[], description?: string): IElementDescriptor;
```

but that's an implementation decision that's outside the scope of this RFC.

### Where does this live?

This would live in a new library, since it defines interfaces for integration between existing libraries, and is not actually dependent on/tied to Ember in any way. The library would export the `ELEMENT_DESCRIPTOR` symbol and types for the `IElementDescriptor` and `IElementDescriptorProvider` interface.

It could also potentially provide other types and helpers, such a `string | Element | IElementDescriptor | IElementDescriptorProvider` type, a `findElement()` method similar to the one in `@ember/test-helpers` that implements most of the element-finding logic in the `click()` example in the [the interfaces](#the-interfaces) section of this RFC, and/or the factory functions described in the [new use cases](#new-use-cases) section, but these are implementation details outside the scope of this RFC.

## How we teach this

This is primarily taught through the API documentation of the various libraries that accept the interfaces, such as `@ember/test-helpers` and `qunit-dom`, and that implement the interfaces, such as `fractal-page-object`. In addition the new library proposed in this RFC would have documentation explaining the interfaces and how to author objects that implement them.

The Ember guides would not need to be updated, as the default Ember testing methodology would be unaffected -- this would be added functionality for users using page object libraries, or implementing ad-hoc element descriptors in their tests.

## Drawbacks

* This would increase the complexity of the DOM helper and assertion APIs
* This would add a small amount of extra maintenance cost to those libraries as all new code does

## Alternatives

### Do nothing

This mainly improves developer ergonomics by allowing for better error/failure messaging and a simpler/more natural semantics when passing page objects to DOM helper/assertion methods. We could decide that the status quo is good enough, and the improved ergonomics are not worth the cost.

### One interface, three symbols

We could get rid of the `IElementDescriptorProvider` interface and modify the `IElementDescriptor` interface to have symbol property keys instead of string property keys. This would simplify the overall picture, but would make implementing ad-hoc descriptors less ergonomic:

```js
let elements = someOtherLibrary.getGraphElements();
let element = elements.length > 0 ? elements[0] : null;
await click({ element, elements, description: 'graph element' });
```

vs.

```js
let elements = someOtherLibrary.getGraphElements();
let element = elements.length > 0 ? elements[0] : null;
await click({ [ELEMENT]: element, [ELEMENTS]: elements, [DESCRIPTION]: 'graph element' });
```

although this would be largely mitigated by the factory functions described in the [new use cases](#new-use-cases) section.

### WeakMap

To address the namespace conflicts in page objects, we could use a WeakMap instead of a symbol and two interfaces. The new library could provide

```ts
const descriptorMap = new WeakMap<object, IElementDescriptor>();

export function registerDescriptorProvider(provider: object, descriptor: IElementDescriptor): void {
  descriptorMap.set(provider, descriptor);
}

export function getDescriptor(provider: object): IElementDescriptor? {
  return descriptorMap.get(provider);
}
```

and page objects (and the like) could register their element descriptors, allowing DOM helpers to retrieve the descriptors when needed.

This would also solve the namespace conflict, but seems to have limited benefits compared to the symbol solution and introduces extra complexity to the solution.

## Unresolved questions

* What do we call the new library?
