---
Start Date: 2017-05-05
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/226
Tracking: https://github.com/emberjs/rfc-tracking/issues/13

---

# Summary

Introduce syntax for passing in multiple named template blocks into a component, and
unify the rendering syntaxes / semantics for
blocks/primitives/component-factories passed into a component.

This RFC is focused chiefly on bringing named blocks to Ember Components,
but it was necessary to define a basic roadmap of functionality for
Glimmer Components as well, but keep in mind that some of the Glimmer-centric details may
change and should hence be considered non-normative.

# Motivation

There are limitations to composition due to the inability to pass more than one block to a component (or 2 blocks if you include the inverse block).

The result of this is that Ember developers have this ultra-powerful, compositional API for overriding portions of a component, but they can only use it in one place in the component invocation; any remaining overrides/configuration needs to be expressed as data and passed in as attributes to the component when it'd be vastly preferable to just pass in a chunk of DOM.

Example:

```html
{{#x-modal headerText=page.title as |c|}}
  <p>Modal Content {{foo}}</p>
  <button onclick={{c.close}}>
     Close modal
  </button>
{{/x-modal}}
```

This works, but the moment you need to render a component in the header (rather than just `headerText`), you end up having to add more config/data/attrs to `x-modal` just to support every one of those overrides, when really you just should be able to pass in another block of DOM to define what the header looks like. The API in this proposal would allow you to express this use case via:

```html
{{#x-modal}}
  <@header as |c|>
    {{page.title}}
    {{status-indicator status=status}}
    {{close-button action=c.close}}
  </@header>

  <@main as |c|>
    <p>Modal Content {{foo}}</p>
    <button onclick={{c.close}}>
       Close modal
    </button>
  </@main>
{{/x-modal}}
```

and with Glimmer components:

```html
<x-modal>
  <@header as |c|>
    {{page.title}}
    {{status-indicator status=status}}
    {{close-button action=c.close}}
  </@header>

  <@main as |c|>
    <p>Modal Content {{foo}}</p>
    <button onclick={{c.close}}>
       Close modal
    </button>
  </@main>
</x-modal>
```

Other RFCs/addons that have attempted to address this:

- [named yields](https://github.com/emberjs/rfcs/pull/72)
- [multiple yields](https://github.com/emberjs/rfcs/pull/43)
- [yet another named yields rfc](https://github.com/emberjs/rfcs/pull/193)
- [ember-block-slots](https://github.com/ciena-blueplanet/ember-block-slots)
- [ember-named-yields](https://github.com/knownasilya/ember-named-yields)
- [local template blocks](https://github.com/emberjs/rfcs/pull/199)

# Detailed design

## Invocation Syntax is separate from Component type

The features specified in this RFC require us to nail down
some specifics as to how Ember and
Glimmer components interop, which syntaxes can be used to render
them, and the mental/teaching model behind how it all works.

- Invocation Syntax (curly vs angle brackets) is conceptually separate from Component
  type (Ember vs Glimmer component)
- "Curly Components" is a misnomer since curly syntax can render both Ember
  components AND Glimmer components
- Angle-bracket syntax can only render Glimmer components
- `{{x-foo @k=v}}` will remain invalid curly syntax due to the `@k=v`
- The way KV pairs provided at invocation is handled depends on the
  component type:
  - Given `{{x-foo k=v}}`, Ember Component `x-foo` will assign/bind `v`
    to property `k` on the component instance, which can be rendered
    within the component layout template as `{{k}}` (same behavior as always).
    `{{@k}}` in an Ember component layout will remain a syntax error
  - Given `{{x-foo k=v}}`, Glimmer Component `x-foo` treats `k=v` as
    assigning/binding arg `@k` to `v`; it will assign/bind `this.args.k`, and expose
    the value as `{{@k}}` within the template. This example invocation
    is functionally equivalent to `<x-foo @k={{v}} />`
  - The mental model is that with curly syntax, `k=v` is the syntax
    for "passing data" to a component; Ember components receive/expose
    this data via the properties on the component instance, and Glimmer
    components receive the data as `@arg`s.
- Curly syntax will not be enhanced with syntax for passing HTML attrs
  (at this time)
- Angle-bracket syntax does not support passing positional params

Implementation-wise, these varying semantics will be defined/implemented via
[Component Managers](https://github.com/emberjs/rfcs/blob/custom-components/text/0000-custom-components.md#componentmanager).

## Multi-block Syntax

Both curly and angle-bracket component invocation syntax will be enhanced
with a nested syntax for passing multiple blocks into a component.

The syntax for curly invocation is as follows:

```html
{{#x-foo}}
  <@header>
    Howdy.
  </@header>

  <@body as |foo|>
    Body {{foo}}
  </@body>
{{/x-foo}}
```

and for angle-bracket invocation:

```html
<x-foo>
  <@header>
    Howdy.
  </@header>

  <@body as |foo|>
    Body {{foo}}
  </@body>
</x-modal>
```

As demonstrated above, the _nested_ syntax for both curly and
angle-bracket multi-block syntax has the format `<@blockName>...</@blockName>`.

This multi-block syntax cannot be mixed with other syntaxes; either ALL
the nested "children" of a component invocation need to be
`<@blockName>...</@blockName>` (multi-block syntax), or none of them do
(classic single-block syntax). The presence of any non-whitespace
character between or around `<@blockName>...</@blockName>`s is a
compile-time error.

Passing two blocks with the same name is a compiler error (though this
might be relaxed in a future RFC).

### Blocks are just (opaque) data

This RFC introduces the concept that blocks are just opaque values
passed into components as data, rather than living in what is
essentially a separate namespace only accessible to `{{yield}}`.

In the above example (with curly syntax), Ember Component `x-foo` would
have its `header` and `body` properties set on its instance. This means,
among other things, that there's no need for a
[hasBlock API for JavaScript](https://github.com/emberjs/rfcs/pull/102);
you can just use normal property lookup / computed properties / etc to
determine whether a block is provided. This means that blocks can
be stashed on services and rendered elsewhere, e.g. the
[ember-elsewhere use case](https://github.com/emberjs/rfcs/pull/226#issuecomment-299949000).

### Unified Renderable Syntax

Rather than continuing to enhance the `{{yield}}` syntax, we should take
this opportunity to unify the various syntaxes for rendering things,
from blocks to primitive values to component factories.

We'll use the following example component invocation to explore
what this syntax looks like: the following (curly syntax) invocation is valid syntax for
rendering either an Ember.Component or a Glimmer Component named
`x-modal` and passing it 3 named blocks: `header`, `body`, and `footer`:

```html
{{#x-modal}}
  <@header as |title|>
    Header {{title}}
  </@header>

  <@body>
    Body
  </@body>

  <@footer>
    Footer
  </@footer>
{{/x-modal}}
```

Given the above invocation, here's how you could render these blocks:

```
{{! within ember-component layout }}
{{this.header "ima title"}}
{{this.body}}
{{this.footer}}

{{! within glimmer-component layout }}
{{@header "ima title"}}
{{@body}}
{{@footer}}
```

Both of these Ember/Glimmer layouts would render:

```
Header ima title
Body
Footer
```

The mental modal here is that is that for ECs, named blocks are
set/bound as a properties on the instance, which we're rendering the
same way we always rendering properties on the instance. For GCs, blocks
are just args that we're rendering with the standard @arg syntax

#### Why `{{this.header}}` and not just `{{header}}`?

Unfortunately, we are constrained by the fact that `{{foo}}` means:

- Try and find a helper named `foo`, and render it
- If no such helper exists, fall back to rendering property `foo`
  on the template context

There is a risk that still exists today that any time you introduce a
new helper to your codebase, for example, a `time` helper, you will break any
templates that try to render a `time` property via `{{time}}`. This is
an unfortunate hangover from the past, and we don't want to continue to
expand the scope of this footgun with this RFC. Hence, to render
blocks/component factories, you must use `{{this.time}}` in your Ember
Component templates.

This is actually an extension to behavior introduced by the
[Contextual Component Lookup
RFC](https://github.com/emberjs/rfcs/blob/master/text/0064-contextual-component-lookup.md);
today, the following is supported:

```hbs
<!-- invocation -->
{{x-foo header=(component 'x-header')}}
```

```hbs
<!-- x-foo layout -->
{{this.header}}
```

In the above example, `{{this.header}}` will actually render the
`x-header` component factory; this RFC merely proposes extending this
syntax so that it can render blocks as well.

Since the potential for forgetting this rule is somewhat high, we
should consider detecting when you use `{{foo}}` syntax when `foo` is a
component factory or a block, and provide a useful warning or error
message to use `{{this.foo}}` instead.

#### Rendering primitives

Consider the following example, which modifies above example by changing
the `footer` block to a string:

```html
{{#x-modal footer="ima footer"}}
  <@header as |title|>
    Header {{title}}
  </@header>
{{/x-modal}}
```

If `x-modal` renders footer via `{{this.footer}}`, then it'll just
render the "ima footer" string just fine; this a nice benefit of having
a Unified Renderable syntax
and supports a common workflow where string args can be promoted
to full-on blocks without having to rework the component code
to support an alternative/parallel API.

#### Call syntax with primitives (or undefined)

If you try to render a block/component with args, e.g. `{{this.foo 1 2 3}}`,
then `foo` MUST be a block or a component. If `foo` is any kind of
non-callable primitive, including undefined, it will be an error.

#### Unified Renderable Syntax: component factories

The following invocation using component factories is also supported:

```html
{{#x-modal
     header=(component 'my-modal-header')
     footer=(component 'my-modal-footer')}}
  <@main>
    Main
  </@main>
{{/x-modal}}
```

(It's worth mentioning that since we're only defining the `main` block
here, this could also be expressed simply as:)

```html
{{#x-modal
     header=(component 'my-modal-header')
     footer=(component 'my-modal-footer')}}
  Main
{{/x-modal}}
```

This demonstrates that the unified renderable syntax is also capable of
rendering component factories (previously only renderable via
`{{component header}}`).

Note since we're passing a positional param `"ima title"` to `header`,
the `my-modal-header` component would only be able to access that param if it were
using the `positionalParams` API with (`reopenClass`), which is a bit of
a clunky / pro-user API.

As a component author, if you want to write your components to support
passing data to both blocks (which accept positional params) and
components (which accept KV pairs), you can pass in both formats
of the same data, e.g.:

```
// Ember.component layout
{{this.header headerTitle title=headerTitle}}

// Glimmer Component layout
{{@header headerTitle title=headerTitle}}
```

#### Named block params

Prior to this RFC, there were only two ways to pass in overridable
chunks of DOM to a component:

- Passing in a block, which only accepts positional block params
  (or a `(hash)` object of named params)
- Passing in a component factory, which only accepts KV args (unless
  you use the `reopenClass-positionalParams` api)

Given that we're introducing a Unified Renderable syntax, it would be
unfortunate if we did nothing to address this impedance mismatch between
named and positional params. The goal is for component consumers/invokers to
be able to pass the most "convenient" kind of Renderable for their use
case, be it a simple primitive string value, a block if they want the
lexical scope + block params, or a component factory for rendering
a shared component that might be used in many places throughout the app.
Unfortunately, the component author will have to choose whether they
want to pass positional params (which would push consumers towards
only using  blocks) or named params (which are presently only supported
by component factories).

Hence, for this reason (among others), it makes sense to introduce a
syntax for named block params; with this syntax, there will be an
organic shift towards component authors using named KV pairs for passing
data in most cases (while still allowing positional params in certain
simpler cases were it only really makes sense to use a block, e.g.
control flow components that wrap `if` or `each`, etc.)

Here is the syntax for named block params:

```html
{{#x-modal}}
  <@header as |@title @data|>
    The title: {{@title}}
    The data: {{@data}}
  </@header>
{{/x-modal}}
```

There is also a "soaking" syntax which is useful in cases where nested
blocks might introduce new named block params that clobber preexisting
identifiers in the scope, as well as cases where spelling out each
named block param consumes too much rightward space. The above example
can be expressed using the soaking syntax as follows:

```html
{{#x-modal}}
  <@header as |@...d|>
    The title: {{d.title}}
    The data: {{d.data}}
  </@header>
{{/x-modal}}
```

(The `@` is not included as part of the identifier as that would
suggest it was a KV arg rather than essentially a hash of args.)

#### Block form of Unified Renderable syntax

It should be possible to pass a block TO the
block/component-factory that's been passed into a component.
The common use cases for this are:

##### Passing a block to a component factory

Given the following invocation:

```
{{x-modal header=(component 'my-header')}}
```

It should be possible for `x-modal` to pass a block to the `header`
renderable:

```
// ember-component x-modal layout
{{#this.header title="ima title"}}
  I'm a block provided by the component layout template.
{{/this.header}}
```

Assuming `my-header` had a layout of:

```
<div class="my-header-inner">
  title is {{title}}
  {{yield}}
</div>
```

This would render the following (assuming `x-modal` and `my-header` are
Ember components with `tagName: 'div'` with `classNames` set):

```
<div class="x-modal">
  <div class="my-header">
    <div class="my-header-inner">
      title is ima title
      I'm a block provided by the component layout template.
    </div>
  </div>
</div>
```

##### Passing a block to a block (aka contextual components)

Given the following invocation:

```
{{#x-modal}}
  <@header as |@title @main|>
    <div class="header-block-content">
      title is {{@title}}
      {{@main}}
    </div>
  </header>
{{/x-modal}}
```

This would render the following (assuming the same `x-modal` layout
as the previous example:

```
<div class="x-modal">
  <div class="header-block-content">
    title is ima title
    I'm a block provided by the component layout template.
  </div>
</div>
```

It would also be possible to pass a component factory to the header
block from `x-modal`'s layout:

```
// ember-component x-modal layout
{{this.header title="ima title"
         main=(component 'x-modal-inner-content')}}
```

The multi-block syntax can be used as well with Unified Renderable
syntax:

```
{{#header}}
  <@main>
    I'm a block provided by the component layout template.
  </@main>

  <@title>
    ima title
  </@title>
{{/header}}
```

##### Block form of `@`-prefixed identifiers

The syntax for passing a block to an `@`-prefixed identifier
(named block params and Glimmer `@arg`s) will be
`{{#@thing}} ... {{/@thing}}`, e.g.:

```
{{#x-layout as |@widget|}}
  {{#@widget as |a b c|}}
    Hi.
  {{/@widget}}
{{/x-layout}}
```

### Classic single-block syntax: `main` and `else` args

It would be unfortunate if component authors had to use different
syntaxes for rendering named blocks vs the traditional "default"
and "inverse" blocks provided by the classic single-block syntax.

Hence, the blocks provided in classic single-block syntax should also
be exposed as properties (Ember) and args (Glimmer), and should have
conventional, meaningful names: instead of "default" (which is a
bit misleading) and "inverse", we standardize on `main` and `else`.

#### Glimmer Components: `@main` and `@else`

Given Glimmer component invocation:

```
{{#fancy-if cond=trueOrFalse}}
  True
{{else}}
  False
{{/fancy-component}}
```

The component layout could be:

```
{{#if cond}}
  {{@main}}
{{else}}
  {{@else}}
{{/if}}
```

Note that angle-bracket syntax doesn't support passing in an
inverse/else block, but the block provided to angle-bracket invocation
would be passed in as `@main`.

#### Ember Components: `main` and `else`

For Ember, we can't suddenly start setting `main` and `else` properties
on the component instances as this would be a breaking change, and
`main` in particular is not an uncommon property name.

We also shouldn't punt on this feature for Ember components for the
following reasons/use cases:

- [ember-elsewhere](https://github.com/emberjs/rfcs/pull/226#issuecomment-299949000)
  (and other similar patterns) require having access to the opaque block
  so that it can be stashed on a service and rendered elsewhere
- wrapper components that forward args/properties/blocks to another
  internal component; blocks need to be accessible as properties in order
  to pass them into another component (otherwise you'd have to use a
  combinatoric mess of block syntax + `if hasBlock` checks to forward
  blocks through to the inner component)

So we need an opt-in API; any Ember Component that wants `main`/`else`
properties to be set on the component instance need to opt into this
behavior via a mixin provided by Ember:

```
import { ImplicitBlockPropertyMixin } from "@ember/implicit-block-property-support";

export default Ember.Component.extend(ImplicitBlockPropertyMixin, {
  blockManager: inject.service(),
  init() {
    this._super();
    this.get('blockManager').registerBlock(this.get('main'));
  },
});
```

So if `fancy-if` were an Ember component that used this mixin, then
given the component invocation:

```
{{#fancy-if cond=trueOrFalse}}
  True
{{else}}
  False
{{/fancy-if}}
```

The following ember component layout would work:

```
{{#if cond}}
  {{this.main}}
{{else}}
  {{this.else}}
{{/if}}
```

# How We Teach This

We teach this as a followup to classic block syntax; once the user is comfortable with single-block syntax, we can introduce named block syntax for more complex use cases.

We teach that what `<@blockName></@blockName>` syntax really means is
that we're just passing in an arg named `@blockName`, which is like
any other arg we might pass into a component, but it happens to point
to a template block than, say, some simple string value.

# Drawbacks

### Different from WC slot syntax

This isn't really anything like the
[WebComponents slot syntax](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/slot)
that intends to address similar use cases, so there is some risk of
introducing an API that doesn't fit in with what the rest of the world
is doing.

### Syntax highlighting changes

Some syntax highlighters might have trouble with this syntax; all
the editors I've tried it on look reasonable, but GitHub's Handlebars
parser isn't too kind (hence I've been using `html` snippets instead
of `handlebars` snippets):

```hbs
<x-modal>
  <@header as |c|>
    ...
  </@header>

  <@main as |c|>
    ...
  </@main>
</x-modal>
```

### Conditionally passing blocks?

This RFC does NOT introduce any kind of facility for conditionally passing
blocks, e.g.:

```html
{{! this syntax is INVALID! }}
<x-layout>
  <@header>...</@header>
  <@main>...</@main>

  {{#if userCanProceed}}
    <@footer>
      {{submit-button}}
    </@footer>
  {{/if}}
</x-layout>
```

This might be desirable in the future, particularly for use cases
involving flex-ish layouts where the component changes behavior /
appearance based on whether blocks on passed in.

# Alternatives

I'd proposed a JSX-y [attr/component-centric](https://github.com/emberjs/rfcs/pull/203) syntax for passing what are essentially DOM lambdas, rendered with `{{component}}`. Perhaps we'll add something like that feature in the future, but it's a much less natural enhancement to Ember than named blocks.

# Considerations for Future RFCs

## Defining Default Blocks

There's not really a nice way defining default blocks inside your
component layout (i.e. the block you render when known is provided at
invocation time), but then again I believe the following would be
a workable approach that is probably support by the features proposed in
this RFC?

```
{{#with-blocks}}
  <@mainOrDefault>
    {{#if main}}
      {{main}}
    {{else}}
      I am the default main block when none is passed in.
    {{/if}}
  </@mainOrDefault>

  <@footerOrDefault>
    {{#if footer}}
      {{footer}}
    {{else}}
      I am the default footer block when none is passed in.
    {{/if}}
  </@footerOrDefault>

  <@render as |@mainOrDefault @footerOrDefault|>
    {{! this specially-named block gets passed all the other blocks above}}

    {{mainOrDefault}}
    {{footerOrDefault}}
  </@render>
{{/with-blocks}}
```

Either way, it feels hacky and weird and I would be surprised if we'd
want/need a future RFC to define a nicer way to support default blocks.

## Allow passing multiple blocks with the same name

e.g.

```html
{{#power-select}}
  <@option value="foo">
    <em>Foo</em>
  </@option>
  <@option value="bar">
    <blink>Bar</blink>
  </@option>
{{/power-select}}
```

This RFC defines that passing multiple blocks with the same name is a
syntax error, but it's something we might want to relax in the future
for certain cases where you want to pass arrays of blocks, such as
`power-select` or use cases involving tables.

