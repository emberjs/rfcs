---
Start Date: 2018-19-12
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/418
Tracking: https://github.com/emberjs/rfc-tracking/issues/30

---

# Deprecate Route render APIs

## Summary

This RFC proposses the deprecation of [`Route#render`](https://emberjs.com/api/ember/3.6/classes/Route/methods/render?anchor=render), [`Route#renderTemplate`](https://emberjs.com/api/ember/3.6/classes/Route/methods/render?anchor=renderTemplate) and named `{{outlet}}` APIs. The following deprecation message will be emitted upon usage of `render` or `renderTemplate`:

```
The usage of `renderTemplate` is deprecated. Please see the following deprecation guide to migrate.
```
and

```
The usage of `render` is deprecated. Please see the following deprecation guide to migrate.
```

The following will be compile time deprecation for named outlets:

```
Please refactor `{{outlet <NAME>}}` to a component <SOURCE_INFO>.
```

## Motivation

These APIs are largely holdovers from a time where components where not as prominent in your typical Ember application. While they are still documented, these APIs created an interesting coupling between the `Route` and the template. These APIs are also prone to breaking conventional naming conventions, which can lead to confusion for developers. Another issue is that it is unclear how something like this works with route based tree shaking, as there are no strong conventions or direct imports as to what is actually being used.

## Transition Path

The migration plan here is going to be somewhat situational based on the UI that was being constructed. For cases where named outlets were being used it is likely that they should just be moved to components. For cases where you were escaping the existing DOM hierarchy to render a template somewhere else in the DOM, one should use the built-in [`{{in-element}}`](https://github.com/emberjs/rfcs/blob/master/text/0287-promote-in-element-to-public-api.md) helper or an addon like [ember-elsewhere](https://github.com/ef4/ember-elsewhere). Below are some example of how a migration would look.

__Migrating Named Outlets__

Given:

```js
Ember.Route.extend({
  // ...

  renderTemplate() {
    this.render('cart', {
      into: 'checkout',
      outlet: 'cart',
      controller: 'cart'
    })
  }
})
```

```hbs
{{! checkout.hbs}}
<section id="items">
  {{outlet}}
</section>
<aside>
  {{outlet "cart"}}
</aside>
```

This would tell Ember to render `cart.hbs` into `checkout.hbs` at the `{{outlet "cart"}}` and use the `cart` controller to back the `cart.hbs` template. This is pretty confusing pattern and creates this implicit coupling that is spread between the `Route` and template.

Luckily, we can express this entire concept with components.

```hbs
{{! checkout.hbs}}
<section id="items">
  {{outlet}}
</section>
<aside>
  <Cart />
</aside>
```

In the event you were using `model` to derive what to render, you can us the `{{component}}` helper to dynamically render a component.

__Migrating Hiearchy Escaping__

Given:

```hbs
{{! app/templates/authenticated.hbs}}

<nav>
  <h1>ACME Corp.</h1>
  <section>
    {{outlet "account"}}
  </section>
</nav>

<section id="content">
  {{outlet}}
</section>
```

```hbs
{{! app/templates/account.hbs }}
{{#link-to 'account'}}
  <img src="{{this.img}}" alt="{{this.name}} />
{{/link-to}}
```

```js
// app/routes/authenticated.js
import Route from '@ember/route';
import { inject as service } from '@ember/service';

export default Route.extend({
  // ...
  user: service('user'),
  renderTemplate() {
    if (this.user.isLoggedIn) {
      this.render('account', {
        into: 'applcation',
        outlet: 'account',
        controller: 'account'
      });
    } else {
      this.transitionTo('login')
    }
  }
});
```

One way this could be migrated is like the following:

```hbs
{{! app/templates/authenticated.hbs}}

<nav>
  <h1>ACME Corp.</h1>
  <section id="account-placeholder"></section>
</nav>

<section id="content">
  {{outlet}}
</section>
```

```hbs
{{! app/templates/authenticated/campaigns.hbs }}

{{outlet}}

{{#in-element this.accountPlaceholder}}
  {{#link-to 'account'}}
    <img src="{{this.account.img}}" alt="{{this.account.name}} />
  {{/link-to}}
{{/in-element}}
```

```js
// app/routes/authenticated.js
import Route from '@ember/route';
import { inject as service } from '@ember/service';

export default Route.extend({
  //...
  user: service('user'),
  beforeModel() {
    if (!this.user.isLoggedIn) {
      this.transitionTo('login')
    }
  }
});
```

```js
// app/controller/authenticated/campaigns.js
import Route from '@ember/route';
import Controller from '@ember/controller';
import { inject as service } from '@ember/service';
import { inject as controller } from '@ember/controller';

export default Controller.extend({
  //...
  account: controller('account')
  init() {
    this._super(...arguments);
    this.accountPlaceholder = document.getElementById('account-placeholder');
  }
});
```

If you want to do this with components you could do the same thing as the following:

```hbs
{{! app/templates/authenticated.hbs}}

<nav>
  <h1>ACME Corp.</h1>
  <section id="account-placeholder"></section>
</nav>

<section id="content">
  {{outlet}}
</section>
```

```hbs
{{! app/templates/authenticated/campaigns.hbs }}
{{outlet}}

<UserAccount />
```

```hbs
{{! app/templates/components/user-account.hbs }}
{{#in-element this.accountPlaceholder}}
  {{#link-to 'account'}}
    <img src="{{this.account.img}}" alt="{{this.account.name}} />
  {{/link-to}}
{{/in-element}}
```

```js
// app/routes/authenticated.js
import Route from '@ember/route';
import { inject as service } from '@ember/service';

export default Route.extend({
  //...
  user: service('user'),
  beforeModel() {
    if (!this.user.isLoggedIn) {
      this.transitionTo('login')
    }
  }
});
```

```js
// app/components/user-account.js
import Component from '@ember/route';
import { inject as controller } from '@ember/controller';

export default Component.extend({
  // ...
  account: controller('account'),
  init() {
    this._super(...arguments);
    this.accountPlaceholder = document.getElementById('account-placeholder');
  }
});
```

# How We Teach This

These APIs not been a mainline API for quite some time now. The guides briefly mention this functionality. In those cases we should mirgate the guides should link to the `{{in-element}}` documentation and the component documentation. The above "Transition Path" will serve as the deprecation guide.

# Role Out Plan

Prior to adding the deprecation we must first do the following items

* [ ] The `{{in-element}}` helper implementation remains incomplete. It should be completed.
  * [ ] The [small changes](https://github.com/emberjs/rfcs/blob/master/text/0287-promote-in-element-to-public-api.md#small-proposed-changes) section of the `{{in-element}}` RFC needs to be implemented. Specifically the helper should "replace all the content of the destination".
  * [ ] The `{{in-element}}` helper should be documented in the [API docs](https://emberjs.com/api/ember/release/classes/Ember.Templates.helpers).
  * [ ] Adding `{{in-element}}` usage to the guides can be considered.

## Drawbacks

The drawback of this is that it is churn for applications that are relying heavily of these imperative APIs to construct their UI. They will need to encapsulate and use the existing declarative APIs.

## Alternatives

No real alternatives as we want to move away from these style of imperative APIs in favor of declarative ones.

## Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?

