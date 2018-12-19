- Start Date: 2018-19-12
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

This RFC proposses the deprecation of [`Route#render`](https://emberjs.com/api/ember/3.6/classes/Route/methods/render?anchor=render), [`Route#renderTemplate`](https://emberjs.com/api/ember/3.6/classes/Route/methods/render?anchor=renderTemplate) and named `{{outlet}}` APIs.

# Motivation

These APIs are largely holdovers from a time where components where not as prominent in your typical Ember application. While they are still documented, these APIs created an interesting coupling between the `Route` and the template. These APIs are also prone to breaking conventional naming conventions, which can lead to confusion for developers.

# Transition Path

The migration plan here is to just move to components for all the use cases you would use theses APIs for. For example, there was a time when you would do something like the following if you wanted to have a master-detail view for a shopping app.

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

# How We Teach This

This has not been a mainline API for quite some time now. The guides do not mention this functionality at all. It is likely that the majority of Ember applications do not use these APIs.

# Drawbacks

The drawback of this is we are saying that the `Route` and the route template of the same name are inherently coupled. In previous versions of Ember this didn't have to be so as long as you implemented `renderTemplate`.

# Alternatives

No real alternatives.

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?
