- Start Date: 2017-01-17
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

There is strong demand for the ability to pass in template blocks into higher components as a means to configure/override how/what they render to DOM, but unfortunately other proposals to that effect have failed to gain traction. This RFC proposes a more general-purpose syntax for defining locally-named template blocks that can be passed into higher order components as attrs.

# Motivation

The need/demand for "named yields"/"block slots" is well-established:

- https://github.com/emberjs/rfcs/pull/43
- https://github.com/emberjs/rfcs/pull/72
- https://github.com/emberjs/rfcs/pull/193
- https://github.com/ciena-blueplanet/ember-block-slots
- https://github.com/ryanto/ember-content-for

In short, people want a way to pass in more than just the default/inverse template blocks, an unfortunate limitation to the flexibility/power of higher order components, and which generally speaking hurts the composability story of Ember components.

The problem with all of the above approaches (aside from the fact that they hack around the lack of such an officially-supported syntax/approach) is that they miss an important and more general use case, which is the ability to define a named local template block and pass it to multiple components, e.g.:

```hbs
{{#let-block sharedHeader as |person|}}
  {{person.name}}
{{/let-block}}

{{complex-component
  header=sharedHeader}}
{{complex-component
  header=sharedHeader}}
{{complex-component
  header=sharedHeader}}
```

None of the proposals I'm aware of support defining `sharedHeader` and passing it to multiple separate components; the most they can do is pass a named block to a higher order component and let the higher order component render it multiple times. It's a subtle but important limitation.

So, given this limitation, and that these proposals have languished, I suggest we ship a more basic / verbose but more fully-featured / flexible syntax today and build a nicer block-slot syntax based off of it in a separate RFC.

# Detailed design

I propose two new additions / primitives to the present-day templating API:

## 1. let-block

`let-block` is a Glimmer built-in that defines a local variable/constant within the current lexical scope that holds a reference to a block that can be rendered multiple times.

```hbs
{{#let-block fooBlock as |a b c|}}
  Hello {{a}}, {{b}}, and {{c}}!
{{/let-block}}

{{x-bar headerBlock=fooBlock}}
{{x-wat headerBlock=fooBlock footerBlock=fooBlock}}
```

`let-block` declarations behave similar to block params in that a reference to `fooBlock` in the above template will always reference the block declared by `let-block` and never try and do a property lookup of `fooBlock` on the context.

`let-block` declarations are hoisted to the top of the current lexical scope, like classic JS function declarations; this means it's possible to reference a block _above_ where it is defined.

```hbs
{{x-bar headerBlock=fooBlock}}
{{x-wat headerBlock=fooBlock footerBlock=fooBlock}}

{{! define fooBlock after it's referenced }}

{{#let-block fooBlock as |a b c|}}
  Hello {{a}}, {{b}}, and {{c}}!
{{/let-block}}
```

## 2. {{yield to=referenceToBlock}}

`let-block` lets you declare blocks, but we still need a way to render them. I propose extending the the `yield` helper syntax:

```hbs
{{#let-block fooBlock as |a b c|}}
  Hello {{a}}, {{b}}, and {{c}}!
{{/let-block}}

{{! render fooBlock here...}}
{{yield 'A' 'B' cValue to=fooBlock}}

{{! ...and again here }}
{{yield 'Ay' 'Bee' 'Cccc' to=fooBlock}}
```

This allows you to render a block whether it's in the same lexical scope, or you've somehow been passed a reference to such a block.

To mimic the behavior of `to="inverse"`, we should probably render nothing if the value passed to `to=` is falsy, but we should probably error if passed an object other than a value produced by `let-block`.

# How We Teach This

I think "named block" is simple and accurate; the only downside is that people familiar with the various RFCs might assume that "named block" refers to specific API for passing named blocks into higher order components, but generally speaking I think it's a new and obvious addition to vocabulary. Other RFCs proposing similar-ish things should probably call what they're providing "slot syntax".

We teach this as a continuation of contextual components and Ember's composability story. This RFC introduces syntax that addresses the heavy requirement that once you go beyond what can be expressed in main/inverse block syntax, you're required to create components/templates that live in separate files, which unnecessarily adds a lot of confusing indirection.

> Would the acceptance of this proposal mean the Ember guides must be re-organized or altered? Does it change how Ember is taught to new users at any level?

This feature could documented in the Guides within or alongside https://guides.emberjs.com/v2.10.0/components/block-params/

# Drawbacks

The biggest drawback is that this proposal is NOT the block slot syntax everyone's been waiting for, and even though it hit more use cases, when applied to the use cases most demanding of block slot syntax, you can wind up with a situation where you declare a bunch of local blocks in a row, and _then_ you pass it to the higher order component, e.g.

```hbs
{{#let-block header as |p|}}
  I am header content {{p}}
{{/let-block}}
{{#let-block footer as |p|}}
  I am footer content {{p}}
{{/let-block}}
{{#let-block body as |p|}}
  I am body content {{p}}
{{/let-block}}
{{complex-component header=header footer=footer body=body}}
```

Most people (including myself) would prefer that in cases where a local block is used only once, it would be better to have a syntax where the blocks are nested in the call to render `complex-component`. The reason I'm OK with this drawback is that, as mentioned earlier, once you want to use a block in multiple places, a syntax specific to block-slots lets you down. It is my hope that another RFC builds off of this one to specify a block-slot syntax that desugars into `let-block` semantics.

The other drawback is that, unlike block params, which are syntactically guaranteed to appear at the top of the scope in which they're introduced, `let-block` syntax doesn't enforce such an order and relies on messy `var`-ish lexical hoisting.

# Alternatives

We could use the classic Handlebars `@data` sigil to name the blocks so that they stick out more.

Another alternative is to say "declaring a local named block and passing it in to multiple places doesn't have strong enough use-cases outside of what could be done if we nailed the block slot syntax". But I worry that those proposals may never land, and I think landing this RFC first would give those proposals a nice target to desugar to.

# Unresolved questions

Is the hoisting behavior a terrible idea? Is there some reason this couldn't work in glimmer?

Would it be possible to pass in a local template block as the `layout` of a component? In such a case, should we support the `yield` helper inside such a template block?