- Start Date: 2015-02-27
- RFC PR:
- Ember Issue:

# Summary

Introduce `Ember.Fragment` and the `<element>` and `<fragment>`
template keywords.

# Motivation

This RFC addresses two key problems:

 - Ember has no clear concept of a "tagless component". We need such a
   concept now that we're moving to a component-centric world.

 - Today users are forced to learn two separate APIs for setting
   element attributes, one that works only for a component's top-level
   element (and cannot be used from inside templates), and one that
   works everywhere else.

# Detailed design

`Ember.Fragment` is introduced to represent "tagless
components". Every Fragment's template must be defined with the
`<fragment>` keyword.

`Ember.Component` continues to *only* represent components with
tags. This is consistent with a web-component world. Every Component's
template **must** be defined with the `<element>` keyword.

## `<element>` semantics by example

If you place this template in `my-component.hbs`:

````handlebars
<element class="special {{class}}" data-my-id={{id}}>
  Hello {{name}}.
</element>
````

It can be invoked like this:

````handlebars
<my-component id=1 name="Tomster" class="magical">
````

And will result in this output:

````html
<div class="special magical" data-my-id="1">
  Hello Tomster.
</div>
````

`<element>` accepts an optional `tagName`, so if we changed the
template to:

````handlebars
<element class="special {{class}}" data-my-id={{id}} tagName="span">
  Hello {{name}}.
</element>
````

The output changes to

````html
<span clas="special magical" data-my-id="1">
  Hello Tomster.
</span>
````

## `<fragment>` semantics

`<fragment>` is very simple, it just declares that your template is a
fragment and not a component. It doesn't generate any output in the
DOM and it doesn't take any arguments. This:

````handlebars
<fragment>
  This is a fragment.
</fragment>
````

Outputs:

````html
This is a fragment.
````

## Invoking Components and Fragments

Components in Ember 2.0 can be invoked with angle bracket syntax:

````handlebars
<my-component>Hola</my-component>
````

But fragments are always invoked with curlies:

````handlebars
{{my-fragment}}Hola{{/my-fragment}}
````

This lets the reader of a template know exactly what they're dealing
with.

## What happens to `attributeBindings`, `classNames` and `classNameBindings`

In the vast majority of cases, users won't need these anymore, and
will be able to get the same control directly from their template
using `<element>`.

However, we will still keep them to handle trickier inheritance
situations, where a base class or mixin needs to control element
attributes in all derived classes. Only library authors are likely to
need to know about them, and most users should be steered toward using
`<element>` instead.

We can also put:

````js
  attributeBindings: ['tagName', 'class']
````
in the base `Component` implementation so that these properties will
propagate from the caller to the `<element>` by default.

Any attributes set directly on `<element>` in a component's definition
**take precedence over** `attributeBindings`. We make a special
exception for `class`: `classNameBindings` and `classNames` will be
merged with any value for `class` provided on the `<element>`. 

## Fragment names

Components need a dash for compatibility with web components. But this
constraint does not apply to fragments, so fragment's names **do not**
need to include a dash.


## Why are `<element>` and `<fragment>` mandatory?

If we want to distinguish between Component templates and Fragment
templates, at least one of them needs to be mandatory. It makes sense
for `<element>` to be the mandatory one, since `<element>` does useful
work and represents a real DOM element.

`<fragment>` is only needed due to upgrade-compatibility
concerns. Some day we'll be able to treat any component without an
`<element>` as a Fragment by default, but right now that would alter
the behavior of existing components.

Therefore, templates with neither `<element>` nor `<fragment>` will
receive legacy behavior.

## Ember.Component and Ember.Fragment classes

Components must extend `Ember.Component`, the same as today.

Fragments must extend `Ember.Fragment`.

Mismatching the class and the template will throw an exception at
instantiation.

`Ember.Fragment` is probably an ancestor of `Ember.Component`, so that
it can define behaviors common to both.

# Drawbacks

Adds a tiny amount of arguable boilerplate to templates that was not
there before.

This proposal doesn't eliminate `attributeBindings`,
`classNameBindings`, and `classNames`. I think it would be a good idea
to eliminate them, but we need a detailed design for some other way to
do inheritance -- one idea would be template macro expansion.

# Alternatives

Instead of `<element>`, components could just use their own real tag
in its place. But this makes it hard to distinguish template
definition from template invocation.

We could avoid introducing `<fragment>` and treat any template without
an `<element>` as a fragment, but this makes the upgrade painful.

We could detect whether a template has a single top-level element or
not and automatically make it by a Component or Fragment. This idea
was rejected because it's a footgun -- for example, accidentally
changing your template from one kind to the other would change the
sematnics of your `$` method.

# Unresolved questions

 - Should `Ember.Component` extend `Ember.Fragment`?

 - Should `Ember.Fragment` have a `$` method that matches its DOM
   range, or should we call it something different to distinguish it
   from the `Component` case, or leave it off entirely?
