- Start Date: 2019-08-05
- Relevant Team(s): Ember.js, Learning
- RFC PR: https://github.com/emberjs/rfcs/pull/523
- Tracking: (leave this empty)

# Provide `@model` named argument to route templates

## Summary

Allow route templates to access the route's model with `@model` in addition to
`this.model`.

Before:

```hbs
{{!-- The model for this route is the current user --}}

<div>
  Hi <img src="{{this.model.profileImage}}" alt="{{this.model.name}}'s profile picture"> {{this.model.name}},
  this is a valid Ember template!
</div>

{{#if this.model.isAdmin}}
  <div>Remember, with great power comes great responsibility!</div>
{{/if}}
```

After:

```hbs
{{!-- The model for this route is the current user --}}

<div>
  Hi <img src="{{@model.profileImage}}" alt="{{@model.name}}'s profile picture"> {{@model.name}},
  this is a valid Ember template!
</div>

{{#if @model.isAdmin}}
  <div>Remember, with great power comes great responsibility!</div>
{{/if}}
```

## Motivation

With the Octane release, templates are taking on a more important role in
idomatic Ember apps. As template become more self-sufficient, many cases where
a JavaScript class was needed in the past (e.g. to customize the "wrapper"
element) can now be expressed with templates alone. This is a direction we will
continue post-Octane.

We would like to update the learning materials (guides) to focus on teaching
the template-centric component model. For example, components can be introduced
as a way to break up large templates into smaller, named pieces, similar to
refactoring a big function into smaller ones. Then, we can layer on making them
reusable through passing arguments. Next, we can introduce the idea of passing
a block and yielding. Finally, we can introduce the component class when we are
ready to discuss adding behavior to the component.

As you can see, we can accomplish quite a lot with templates-only components in
Octane, and focusing on the teaching template first-and-foremost would provide
a gentle learning curve for developers and designers who are comfortable with
HTML but perhaps new to Ember (or even JavaScript). With this flow, the concept
of a component class, and consequently, the idea of a "`this` context" in a
template comes up quite late.

This presents a problem, as we may want or need to teach route templates before
that was introduced. As the model can only be accessed through `this.model` in
a route template, we would be forced to introduce that concept (and the contept
of a controller) at an earlier time than would be ideal.

Providing `@model` to route templates would solve this problem quite elegantly.
We will now be able to introduce the concept of arguments (`@name`) once and it
can be applied consistently between component and route templates.

This also aligns with the general mental model that arguments are things that
are passed into the template from the outside (which is true in the case of the
route model).

This can also be thought of as a small incremental step in the bigger picture
of reforming route templates and removing controllers from Ember. Specifically,
it moves us a bit closer to the mental model that controllers/route templates
are just a "special" kind of component. We expect to continually unify them and
remove the remaining differences, and this is a step towards that direction.

## Detailed design

Internally, route templates are _already_ modelled as components at the Glimmer
VM layer. To implement this, we would "synthesize" a named argument `@model`
containing the resolved route model, i.e. the same value as `this.model` on the
controller instance.

Just like the "reflected" named arguments in classic components, mutating
`this.model` on the controller instance would _not_ change the value of
`@model`. In practice, this seems unlikely to be relied upon and probably
considered a bad practice (it does not change the URL, does not affect what is
returned by `route.modelFor`, etc). In any case, this is consistent with the
general behavior for named arguments, in that they are immutable and should
always reflect what was "passed in" from the caller.

## How we teach this

* `@model` is passed into the component from the route
* `this.model` is how you access the same thing from the controller, if needed
* Guides and API docs should be updated to use `@model` instead of `this.model`

## Drawbacks

None?

## Alternatives

We can do nothing.

## Unresolved questions

None?
