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

[@ember/test-helpers](https://github.com/emberjs/ember-test-helpers)' DOM helpers and [qunit-dom](https://github.com/simplabs/qunit-dom) accept either a selector or an HTMLElement as their DOM element descriptor, which is a pretty common pattern. This RFC proposes an interface that generalizes the notion of a DOM element descriptor that `@ember/test-helpers`' DOM helpers, `qunit-dom`, and other similar test support libraries in the Ember ecosystem can support, so page object implementations such as [fractal-page-object](https://github.com/bendemboski/fractal-page-object) and [ember-cli-page-object](http://ember-cli-page-object.js.org/) can implement the interface and integrate cleanly and ergonomically with these test support libraries.

## Motivation

Ember's testing ecosystem has converged on a single set of recommended DOM APIs. `@ember/test-helpers` provides the core value of implementing DOM interactions, while `qunit-dom` provides the core value of making assertions about DOM query results. They do _not_ (nor should they) provide the core value of advanced DOM querying -- they act on DOM elements, and do the simple baseline of supporting passed selectors and HTMLElements as descriptors of the DOM element(s) to act on.

[Page Objects](https://martinfowler.com/bliki/PageObject.html) are a powerful and popular tool for making test code well organized, readable, and maintainable. The core value they provide is translating between a particular page's (or fragment of a page's or component's) rendered DOM and a functional/logical description of the page/fragment/component, in other words a page-specific advanced DOM query language. Supporting a clean integration of page objects with libraries like `@ember/test-helpers` and `qunit-dom` that act on DOM element descriptors would keep each focused on its core value, while supporting page objects as "first-class" DOM element descriptors.

Currently `fractal-page-object` integrates with `@ember/test-helpers` and `qunit-dom` by exposing an `element` property on all page objects, and that works because their APIs accept HTMLElements. But using HTMLElements as descriptors has three major drawbacks compared to using selectors:

1. It only supports single elements. While this isn't an issue with the `@ember/test-helpers` DOM APIs, in `qunit-dom` that means there's no way to use the multi-element assertions, like `assert.dom(document.querySelectorAll('.list-item')).exists({ count: 3 })`.
2. It does not produce a useful information when encountering an error/failure. For example, `await click('.does-not-exist')` errors with `Error: Element not found when calling click('.does-not-exist').`, while `await click(document.querySelector('.does-not-exist'))` errors with the less helpful `Error: Must pass an element or selector to 'click'` (because it doesn't have any other information to show -- all it knows is it was passed `null` instead of a selector or HTMLElement). Similarly, `assert.dom('.does-not-exist').exists()` produces the assertion message `Element .does-not-exist exists`, while `assert.dom(document.querySelector('.does-not-exist')).exists()` produces the less helpful `Element <not found> exists`.
3. It supports a much less ergonomic integration with page objects. In `fractal-page-object`, the integrations currently look like `await click(page.listItems[2].checkbox.element)` and `assert.dom(page.listItems[2].checkbox.element).isChecked()`, rather than `await click(page.listItems[2].checkbox)` and `assert.dom(page.listItems[2].checkbox).isChecked()`. While it may look like a minor difference, in practice it's quite a stumbling block as the mental model of page objects encourages users to think of page object nodes (e.g. `page.listItems[2].checkbox`) as a proxy for the DOM element(s) they describe, so forgetting to add the `.element` is extremely common, even for the author of the library!

We can solve these by defining an DOM element descriptor interface that allows page object implementations, or even one-off test helpers written by application authors, to implement objects that can be passed to these test helpers and support everything that the selector API supports, specifically multiple elements and an optional description of the result set for messaging/debugging, and all without sacrificing developer ergonomics.

## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
