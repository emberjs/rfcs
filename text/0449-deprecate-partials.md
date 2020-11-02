---
Start Date: 2019-02-17
Relevant Teams: Ember.js, Learning
RFC PR: https://github.com/emberjs/rfcs/pull/449
Tracking: https://github.com/emberjs/rfc-tracking/issues/38

---

# Deprecate `{{partial}}`

## Summary

Partials are an old Ember construct that have no benefits and many downsides when compared to modern Ember components. We should deprecate them in favor of components.

## Motivation

Partials have a number of downsides when compared with components: 

 - They are hard to reason about as they inherit the scope of the calling template
 - They perform poorly in comparison to components

Partials should have no place in modern Ember applications, components should always be preferred.

Once removed, Ember's API will become smaller and more consistent.

## Detailed design

We'll create an AST transform in [`packages/ember-template-compiler`](https://github.com/emberjs/ember.js/tree/master/packages/ember-template-compiler) which will emit a deprecation warning for all uses of `{{partial}}`. The deprecation warning will be:

```
Using `{{partial}}` is deprecated, please use a component instead.
```

This message will link to the following deprecation details which aim to give clear guidance on how to migrate to component:

---

The use of `{{partial}}` has been deprecated, you should replace the partial with a component as follows:

Let's consider a simple example of a `partials/quick-tip` partial being invoked from the `application.hbs` template:

`app/templates/application.hbs`

```hbs
{{#let (hash title="Don't use partials" body="Components are always better") as |tip|}}
  {{partial "partials/quick-tip"}}
{{/let}}
```

`app/templates/partials/quick-tip.hbs`

```hbs
<h1>Tip: {{tip.title}}</h1>
<p>{{tip.body}}</p>
<button {{action 'dismissTip'}}>OK</button>
```

Here's the same template code after migrating the `partials/quick-tip` partial to be a component.

`app/templates/application.hbs`

```hbs
{{#let (hash title="Don't use partials" body="Components are always better") as |tip|}}
  <QuickTip @tip={{tip}} @onDismiss={{action 'dismissTip'}} />
{{/let}}
```

`app/templates/components/quick-tip.hbs`

```hbs
<h1>Tip: {{@tip.title}}</h1>
<p>{{@tip.body}}</p>
<button {{action @onDismiss}}>OK</button>
```

---

A codemod, while not necessary for this RFC to land, would greatly simplify the migration of partials to components. We should endeavor to create this codemod as part of efforts to implement this RFC. The codemod might work as follows:

 * Examine each partial template to recognize which properties originate from the caller scope
 * Generate a component to replace each partial
 * Replace each `{{partial "foo-bar"}}` invocation with a component invocation, passing arguments as required
 * Delete the partial handlebars files

 If we do implement a codemod, we should mention it in the deprecation details above.

## How we teach this

We'll mentiton the deprecation in an Ember point release blog post.

The deprecation message will contain a link to clear guidelines on how to migrate from a `{{partial}}` to a component. See the section above beginning with "The use of `{{partial}}` has been deprecated..." for the content.

There are no changes to make to the Ember.js guides, since mentions of `{{partial}}` were removed in 2.x guides.

## Drawbacks

This adds a little churn to Ember's API.

## Alternatives

We could do nothing and leave things as is.

## Unresolved questions

None
