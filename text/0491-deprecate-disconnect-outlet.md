---
Start Date: 2019-05-20
Relevant Team(s): Ember.js, Learning
RFC PR: https://github.com/emberjs/rfcs/pull/491
Tracking: (leave this empty)

---

# Deprecate `disconnectOutlet`

## Summary

This RFC deprecates `Route#disconnectOutlet` because it has no use case given [#418](https://github.com/emberjs/rfcs/pull/418).

## Motivation

`Route#disconnectOutlet` is intended to be used in conjunction with `Route#render`. When we wrote [#418](https://github.com/emberjs/rfcs/pull/418) we should have also deprecated `Route#disconnectOutlet` because it is primarily used to teardown named outlets setup by `Route#render`. For all intensive this is just an addendum to #418.

## Transition Path

The transition path is the same to the one outlined in [#418](https://github.com/emberjs/rfcs/blob/master/text/0418-deprecate-route-render-methods.md#transition-path). Since the migration path for the named outlets is to use components, a developer would need to wrap the component in a conditional if they want to control the destruction.

Given:

```js
// app/routes/checkout.js

class CheckoutRoute extends Route {
  // ...

  @action
  showModal() {
    this.render('modal', {
      outlet: 'modal',
      into: 'application'
    });
  }

  @action
  hideModal() {
    this.disconnectOutlet('modal');
  }
}
```

```hbs
{{! app/templates/checkout.hbs}}

<button {{action 'showModal'}}>Show Modal</button>
<button {{action 'closeModal'}}>Close Modal</button>
```

```hbs
{{! app/templates/application.hbs}}
{{outlet "modal"}}

<main>
  {{outlet}}
</main>
```

This can transitioned to:

```js
// app/controller/checkout.js

class CheckoutController extends Controller {
  // ...
  @tracked isModalOpen = false;

  init() {
    super.init();
    this.modalElement = document.getElementById('modal');
  }

  @action
  showModal() {
    this.isModalOpen = true;
  }

  @action
  closeModal() {
    this.isModalOpen = false;
  }
}
```

```hbs
{{! app/templates/checkout.hbs}}

<button {{action 'showModal'}}>Show Modal</button>
<button {{action 'closeModal'}}>Close Modal</button>

{{#if this.isModalOpen}}
  {{#in-element this.modalElement}}
    <Modal />
  {{/in-element}}
{{/if}}
```

```hbs
{{! app/templates/application.hbs}}
<div id="modal"></div>

<main>
  {{outlet}}
</main>
```

The above example will conditionally append the modal component into `div#modal` whenever the user toggles the modal.

## How We Teach This

Once deprecated, developers will be presented with the following deprecation warning:

```
"disconnectOutlet" has been deprecated for disconnecting outlets.
```

This deprecation message will also link to the transition guide. The transition guide will cover how to migrate named outlets to components. In addition, the guides should be updated to remove any usage of these APIs.

## Drawbacks

N/A. This addendum to [#418](https://github.com/emberjs/rfcs/pull/418).

## Alternatives

N/A. This addendum to [#418](https://github.com/emberjs/rfcs/pull/418).

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
