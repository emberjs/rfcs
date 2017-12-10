- Start Date: 2017-12-10
- RFC PR: 
- Ember Issue: 

# Summary

A proposal to allow the `link-to` helper to expose the active state via
`yield`. Doing so can allow more flexable use cases and DOM layouts.

# Motivation

link-to will add an `active` class to its element when the route matches.
However, many CSS frameworks or addons require a more specific layout where the
inner child needs the `active` class but the `onclick` event needs to be on the
parent link-to element. For example Bootstrap CSS requires the child element to
have the `active` state to render correctly but the parent element mush have the
onclick event handler otherwise the target area when rendered does not cover the
entire UI (due to margins/padding).

The link-to also knows how to generate a URL (href) based on the route name.
Doing this conversion manually each tim emeans the unsertanding of route name to
URL will be in multiple places in the app when instead could be calculated if
exposed by the link-to helper.

# Detailed design

The syntax for link-to follows the convention of yielding to the consumer. It is
expected that clicking the link-to element will perform the route transition and
when the target route matches the current route the same element recieves the
`active` class. The content of the element rendered to the DOM is often provided
by the consumer as yielded content.

```hbs
{{#link-to "my-route" tagName="li"}}
  <a href="/hard/coded">Yielded Content</a>
{{/link-to}}
```

Which renders as:

```html
<li class="active">
  <a href="/hard/coded">Yielded Content</a>
</li>
```

By yielding the active and href state we can leverage the link-to logic in the
context of our own code:

```hbs
{{! Example 1 }}
{{#link-to "my-route" tagName="li" as |link|}}
  <a href={{link.href}} class="{{if link.active 'active'}}">My Route</a>
{{/link-to}}

{{! Example 2 }}
{{#link-to "my-route" tagName="li" as |link|}}
  {{#if link.active}}
    <span class="current-route red">Current Route</span>
  {{else}}
    <a href={{link.href}}>My Route</a>
  {{/if}}
{{/link-to}}
```

# How We Teach This

The documentation for link-to will change the yield example to include a yielded
variable called `link` which will have two properties `link.active` and
`link.href`. A paragraph to explain that meaning of them and an example (see
above) to illistrate a possible use case.

Since this feature adds functionality to an existing API the current guides on
the link-to helper are enough as they are since it covers most use cases. When a
user needs more information (such as looking up `tagName` and `class`) via the
API section they can also discover the use of this feature.

This change would **not** modify how Ember is tought. It is backwards
compatible with current practises and typical usage out in the wild. The added
functionality need only documented once in the API for link-to.

Most existing Ember users are familure with the use of yielded variables and
adding this would not need much education beyond just letting them know it is
available via a ChangeLog and in the API documnetaion for the link-to component.

# Drawbacks

A tradeoff is that the two exposed values now become public. Which means it does
add the slight overhead to maintaining this part of the code since once made
public it will require more scrutiny before changing.

# Alternatives

This section could also include prior art, that is, how other frameworks in the
same domain have solved this problem.

Per this [Stack Overflow question][1] you can nest the link-to twice to cover
teh specific Bootstrap problem. Unfortunatly, it does not offer the flexability
this feature allows for more advanced use cases.

The same [Stack Overflow question][1] also has a sugestion of wrapping the
link-to in a custom component which can investigate its DOM for active elements:

```js
export default Ember.Component.extend({
  tagName: 'li',
  classNameBindings: ['active'],
  active: Ember.computed('childViews.@each.active', {
    get() {
      return this.get('childViews').anyBy('active');
    }
  })
});
```

The disadvantage here is that the onclick handler is not on the parent element
and can cause styling problems if that requirement is needed. **This alternative
is no loger available since use of Glimmer.**

[1]: https://stackoverflow.com/questions/14328295/how-do-i-bind-to-the-active-class-of-a-link-using-the-new-ember-router

# Unresolved questions

1.  What other drawbacks exists?
2.  Are there other alternatives / workarounds that haven't been explored in
    this RFC?
3.  What other use cases should be documented in this RFC?
