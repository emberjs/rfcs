- Start Date: 2017-01-18
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

This RFC proposes an extension to the Handlebars/Glimmer syntax to support passing template blocks into components as attributes, as well as standardizing on a "component-centric" (rather than "yield-centric") approach to rendering multiple template blocks passed into a component.

# Motivation

The [need/demand](https://github.com/emberjs/rfcs/pull/43) [for](https://github.com/emberjs/rfcs/pull/72) ["named yields"](https://github.com/emberjs/rfcs/pull/193)/["block slots"](https://github.com/ciena-blueplanet/ember-block-slots) [is](https://github.com/knownasilya/ember-named-yields) [well-established](https://github.com/emberjs/rfcs/pull/199); we need a nice syntax for passing multiple chunks of DOM/logic into a component. Today, the following options are available:

1. Use the block + block params syntax, and `{{yield}}` within the component.
	- Conveniently closes over parent scope; blocks can access anything the parent can, plus block params
	- Only supports passing in main / inverse template
2. Pass `attr=(component 'x-foo' values=...)` into the component, and `{{component attr}}` within the component.
	- No limit to the number of `(component)`s that can be passed in
	- No closed-over-scope; all required data must be explicitly passed into `(component)` as attrs
	- File bloat occurs since single-use components must live in separate .js/.hbs files

Both of these approaches are well-established/documented/blogged/understood in the Ember community, but what's missing is a syntax that supports the combined/best qualities of the above:

1. Closed over scope + block params
2. Support for passing in multiple/unlimited template blocks
2. Minimal file bloat

The various slot/"named yield" RFCs attempt to specify a syntax with these qualities, but suffer similar issues:

1. The behavior/syntax surrounding the default block is unusual/bizarre when all you want to specify are named blocks
2. The enhancement of the `{{yield}}` API poses compatibility / composability issues and fragments the component ecosystem

After talking over my [previous attempt at an RFC](https://github.com/emberjs/rfcs/pull/199) with the esteemed `#topic-forms` gang in the Ember Community Slack, it seemed more and more like slots/named yield syntax was more of a scenario solve than a universal, robust solution to this problem; providing a solution specific to passing multiple template blocks into a component would have left other similar use cases unsolved.

The syntactic enhancements proposed in this RFC were ultimately inspired by (hold your breath) JSX. One thing JSX does right is that it allows recursive nesting of JSX within JS within JSX within JS, and so on. For example, here is a snippet of JSX that nests multiple "layers" of JS and JSX:

```
<MediaPlayer
  tracklistComponent={ data =>
    data.tracks.map(track =>
      <div className="track">{ track.name }</div>
    )
  }
/>
```

`{ ... }` denotes regions of JavaScript, whereas `<...>` denotes regions of JSX, and there is no restriction as to whether/where JSX can be nested in JS, and vice versa. It is _this_ particular property of JSX that I am proposing we bring to Glimmer template syntax.

Glimmer/Handlebars syntax requires that all DOM syntax exist at the top, un-nested level. Once you're inside some curly braces (`{{...}}`), you're stuck in Handlebars-land until the closing curly braces. I propose we embrace a syntax that makes it easy to pass in template blocks as attrs to a component. Something _like_ the following:

```
// classic curly style:
{{media-player
    tracklist=(|data|
      {{#each data.tracks as |t|}}
        <div class="track">{{ track.name }}</div>
      {{/each}}
    )
}}

// angle-bracket component style:
<media-player
    @tracklist={{|data|
      {{#each data.tracks as |t|}}
        <div class="track">{{ track.name }}</div>
      {{/each}}
    }}
/>
```

I also propose that the manner in which you _render_ a reference to a template block is via `{{component tracklist}}` rather than extending the `{{yield}}` API (as most people are probably expecting). The reason for this is nuanced but I argue it will result in the most cohesive API for using/passing/composing components than a yield-centric approach.

# Detailed design

## Recursively-Nestable Glimmer/Handlebars syntax

_HELP WANTED: For this first version of the RFC, I am not prepared to recommend a specific syntax and would welcome the help of any Glimmer contributors who might help narrow down the particulars_

The grammar for Glimmer/Handlebars needs to be amended to allow for recursively nest-able DOM/Glimmer syntax (see above for a rough example).

1. There must be no constraints on how deeply DOM/Glimmer syntaxes can be nested. In other words, it should be possible to express the following:

```
{{x-foo
    header=(|d|
      {{my-header
        subheader=(|h|
          <h1>{{d.foo}}, {{h.foo}}, and {{outerFoo}}</h1>
        )
      }}
    )
}}
```

2. Scoping rules must behave the same as classic yield+block params. In the above example, all of the template blocks (`header` and `subheader`) can access the outside scope (e.g. `{{outerFoo}}`) along with any block params passed to them.

3. Unwrapped DOM fragments must be supported, e.g.:

```
{{x-foo
    header=(|d|
      I am a fragment {{d.title}}
      <span>Woot</span>
    )
}}
```

## Render blocks with `{{component}}`

Continuing with the example above, if we were in `x-foo`'s layout template and we wanted to render the `header` attr passed into us, we would use the `{{component}}` helper:

```hbs
{{component header title="Hello"}}
{{component header title="Another Header?"}}
{{component header title="Why not..."}}
```

The `{{component}}` helper (and its `(component)` form) would need to enhanced as follows: when passed a template block, render it as a fragment (i.e. `tagName: ""`), and pass it a single block param of all the attrs passed to the `component` helper (e.g. `title="Hello"`).

## Why not `{{yield}}`?

The "component-centric" API proposed above might come as a shock to people expecting to use `{{yield to=header}}` (or some equivalent extension to `yield` to render blocks.

Unfortunately, if we were to take the "yield-centric" approach, we would actual cause great harm to our component API and ecosystem.

### Why not `{{yield}}`: Impedance Mismatch with `(component)`

Consider today's API for passing in multiple snippets of DOM/logic into a component:

```hbs
{{x-foo
    header=(component 'x-header' x=1)
    body=(component   'x-body'   y=2)
    footer=(component 'x-footer' x=3) }}
```

This is and will continue to be an extremely powerful, flexible, and composable API. Even with the ability (as proposed in this RFC) to pass multiple blocks into a component, there'd still be a need for the above API a) whenever the components need to be used in multiple places and b) whenever the template block grows so large that it'd be easier to understand as a separate file.

As such, it is extremely important that a component's consumer be able to swap between passing in a `(component ...)` and a template block without the component's internals needing to change in any way. There is no good/clean way to meet this constraint if we take a yield-centric approach.

### Why not `{{yield}}`: Positional Params vs Named Attrs

The source of the problem is how data is passed: `yield` passes positional block params, while `component` passes named attrs (the esoteric `positionalParams` API is not a solution here).

But, you ask, couldn't we still pass in template blocks that accept multiple positional block params, and do the block param / attr translation when we render the component? e.g.

```hbs
{{x-foo
    header=(|foo bar|
      {{my-header title=foo theme=bar}}
    )
}}
```

First off, this assumes that we don't care that people have already written plenty of components in the `header=(component ...)` style; those people would have to rewrite both `x-foo`'s internals and change any code that renders `x-foo` to support this.

Secondly, this "translation layer" wouldn't just be necessary in this one place, but would be necessary anytime you wanted to pass this template along to someone expecting a component. Perhaps `x-foo` actually passes `header` to another, internal `super-x-foo-header` component to render the header, and _that_ component expects to be passed something as a `(component)` so that it might pass named attrs to it when it renders. You'd end up with translation layers all over the place, all because of the chasm between yield-centric and component-centric rendering.

But, you ask, what if we adopted a pattern of `{{yield (hash title="..." value=123) to=header}}`, such that we only pass a single hash to the template block? This _might_ technically work but it's a super boilerplate-y pattern to remember.

### Why not `{{yield}}`: Hiding useful data considered harmful

I am highly critical of APIs that are Handlebars-specific, such that a piece of data is only accessible/bindable/passable via a Handlebars helper for which there is no JavaScript equivalent. `{{hasBlock}}` is an example of such an API, and as will always be the case for any Handlebars-only API, its lack of a JavaScript equivalent [thwarts unforeseen but perfectly reasonably use cases](https://github.com/emberjs/ember.js/issues/11741).

Generally speaking, the API for yielding blocks in Ember lives in its own world and plays according to its own rules; the blocks provided aren't introspectable as attributes, but rather require block-specific Handlebars API for introspecting. Ultimately, these separate-world semantics hurt compatibility/composability with the rest of the Ember ecosystem that can only be patched over with new APIs (and likely-to-languish RFCs). Want to write a computed property based on whether a block was passed? [Submit an RFC](https://github.com/miguelcobain/rfcs/blob/js-has-block/text/0000-js-has-block.md). Want to "capture" a block in a layout template and forward it to another component to wrap/render? We'd need an RFC to add `(yield)` with similar semantics as `(component)`.

The `attr=(component ...)` pattern, on the other hand, doesn't suffer the same limitations and it composes extremely well with both JS and Handlebars because it just uses attrs/properties which work and have meaning _everywhere_ in Ember. And we should continue to standardize on it rather than opting to further enhance the `{{yield}}` API.

_FWIW: The sentiments expressed here echo those of the [Extensible Web Manifesto](https://github.com/extensibleweb/manifesto)._

### Why not `{{yield}}`: Conclusion

It is unfortunate that `{{yield}}` is not the solution, since that is likely what everyone is expecting, and it might have been easier to teach, but if we were to continue to build off of it, we'd be introducing massive amounts of impedance mismatch between component/yield-based APIs that would require lots of translation layers and "ugh do I really have to" boilerplate APIs.

There may be arguments founded in performance considerations as to why `{{yield}}` might be preferable to build on top of, but I think a solid case has been made that the composability losses would outweigh the performance considerations, and that we should work to optimize the most consistent/composable API, than shift the API towards something that's fast but finnicky.

# How We Teach This

One major challenge is that we'd need to explain when/if to use `{{yield}}` when building components. _I will fill out this section more once general feedback starts to roll in for this RFC_

Generally speaking though, we would teach this feature after teaching the syntax we have today for passing in multiple components:

```hbs
{{x-foo
    header=(component 'x-header' appName=appName)
    body=(component   'x-body'   text=text)
    footer=(component 'x-footer' appName=appName) }}
```

We'd point out that while the above approach works (and is the best option for when `x-header/footer/body` are used in other parts of the app), it is a heavy solution that involves multiple files and manually passing data between them, and that there is an alternative that works wonders and can be gradually introduced where you see fit:

```hbs
{{x-foo
    header=(component 'x-header' x=1)
    body=(component   'x-body'   y=2)
    footer=(
      <div>I'm the footer for {{appName}}</div>
    )
}}
```

Also, (and I think this has come up in the past), we might want to consider generalizing our definition of a "component" to any chunk of DOM with optional JS code to drive it. This might alleviate some of the strangeness of rendering a block template with `{{component}}`, and would also serve as replacement terminology when we phase out "partials".

# Drawbacks

- Syntax
	- The recursively nestable syntax will be a surprise/shock to some/many.
	- The changes will likely require nontrivial updates to syntax highlighters. [Here's how my Vim setup parses the strawman syntax](http://machty.s3.amazonaws.com/to_s3_uploads/Pasted%20image%20at%202017_01_20%2012_48%20PM.png)
- We'd effectively be putting the brakes on `{{yield}}` as the favored pattern for rendering passed-in chunks of DOM
- The fact that the single block param passed to a template block is a hash of attrs means that given block param `|v|` all references to the values passed will need to be prefixed with `v.`, which is uglier than just using positional params and referring to those params by name. (Though perhaps some day this might be alleviated by some future destructuring syntax).

# Alternatives

Alternatives include the other RFCs for slots/named yields:

- https://github.com/emberjs/rfcs/pull/43
- https://github.com/emberjs/rfcs/pull/72
- https://github.com/emberjs/rfcs/pull/193

Also, if the syntactic enhancements to Handlebars/Glimmer are too harrowing, we could consider a lighter-weight (and less universal) approach for collecting anonymous blocks and passing them into a component as attrs using the `#with-slots` approach described in [this gist](https://gist.github.com/machty/c00c83f3fbefefa72cefb0bb2322fc4f#component-centric-slot-syntax).

One possible extension to this RFC: it might makes sense for the block param passed to the block template to be an instance of Ember.Component generated by the `{{component}}` helper that rendered the block. This would only be useful if we allowed specifying a component class when "casting" the template block to a component, such that the component class could provide/expose computed properties, services, actions, tasks, etc.

# Unresolved questions

TODO: need to nail down a specific workable nested Glimmer syntax.

The features in this RFC, in conjunction with something like [inline-let](https://github.com/emberjs/rfcs/pull/200) (or even just the `with` helper) might alleviate some pathological cases of repetition that occur when a chunk of template code is conditionally nestable. One example is in [Liquid Fire](https://github.com/ember-animation/liquid-fire/blob/20613cb/addon/templates/components/liquid-if.hbs). You could imagine this being solved via something like:

```
{{let template=(|a|
  {{#liquid-versions ... notify=a.notify}}
    ...
  {{/liquid-versions}}
)}}

{{#if containerless}}
  {{component template}}
{{else}}
  {{#liquid-container ... as |container|}}
    {{component template notify=container}}
  {{/liquid-container}}
{{/if}}
```

This usage of `{{component}}` definitely strikes me as super weird at first, and only starts to make more sense if we gradually migrate our terminology to mean "a chunk of DOM and maybe code". But perhaps this is an argument in favor if still making it possible to use `{{yield}}` in controlled cases?

# Appendix: Examples

## Example: Recursive Nesting

```
{{x-foo
    header=(|a|
      I am a fragment {{a.title}}
      <span>Woot</span>
    )
    body=(|a|
      I am a fragment {{a.title}}
      <span>Woot</span>

      {{x-foo
          header=(|b|
            I am a fragment {{b.title}}
            <span>Woot</span>
          )
          body=(|b|
            I am a fragment {{b.title}}
            <span>Woot</span>
          )
          footer=(|b|
            I am a fragment {{b.title}}
            <span>Woot</span>
          )
      }}
    )
    footer=(|a|
      I am a fragment {{a.title}}
      <span>Woot</span>
    )
}}
```

## Example: Addon `app/` tree override patterns

This example demonstrates how the `{{component}}`-centric approach fits in nicely with established ember-cli addon conventions for overrides (where as the `yield`-centric approach would need new/additional conventions for such a thing):

Consider the following layout template for an `x-calendar` component that ships with an Ember addon:

```
{{component 
   (or header 'x-calendar-default-header')
   headerText=headerTextFromXFoo}}

{{#each days as |day|}}
  {{component 
      (or dayCell 'x-calendar-default-day-cell')
      day=day otherData=otherData}}
{{/each}}

{{component 
   (or footer 'x-calendar-default-footer')
      footerText=footerTextFromXFoo}}
```

One nice thing thing here is you're provided with two ways of overriding the default appearance of `x-calendar`:

- The `x-calendar` addon presumably ships default components .js and .hbs files in the `app/` tree, so if you want to override a component for the whole app, you can just do the typical ember-cli approach of overriding that hbs/js file in your `app/` folder
- If you plan to render multiple `x-calendar`s that each have their own appearance/overrides, you can pass in those overrides as attrs, e.g:

```
{{x-calendar events=events
     header=(component 'my-header-override-1' style='tomato')}}
```

or using the nested template style in this RFC:

```
{{x-calendar events=events
     header=(|d|
       <h1 class="tomato-header">{{d.title}}</h1>
     )
}}
```









