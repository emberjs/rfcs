- Start Date: 2018-03-09
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Angle Bracket Invocation

## Summary

This RFC introduces an alternative syntax to invoke components in templates.

Examples using the classic invocation syntax:

```hbs
{{site-header user=this.user class=(if this.user.isAdmin "admin")}}

{{#super-select selected=this.user.country as |s|}}
  {{#each this.availableCountries as |country|}}
    {{#s.option value=country}}{{country.name}}{{/s.option}}
  {{/each}}
{{/super-select}}
```

Examples using the angle bracket invocation syntax:

```hbs
<SiteHeader @user={{this.user}} class={{if this.user.isAdmin "admin"}} />

<SuperSelect @selected={{this.user.country}} as |Option|>
  {{#each this.availableCountries as |country|}}
    <Option @value={{country}}>{{country.name}}</Option>
  {{/each}}
</SuperSelect>
```

## Motivation

The original [angle bracket components](https://github.com/emberjs/rfcs/pull/60)
RFC focused on capitalizing on the opportunity of switching to the new syntax
as an opt-in to the "new-world" components programming model.

Since then, we have switched to a more iterative approach, favoring smaller
RFCs focusing on one area of improvment at a time. Collectively, these RFCs
have largely accomplished the goals in the original RFC without the angle
bracket opt-in.

Still, separate from other programming model improvements, there is still a
strong desire from the Ember community for the previously proposed angle
bracket invocation syntax.

The main advantage of the angle bracket syntax is clarity. Because component
invocation are often encapsulating important pieces of UI, a dedicated syntax
would help visually distinguish them from other handlebars constructs, such as
control flow and dynamic values. This can be seen in the example shown above –
the angle bracket syntax made it very easy to see the component invocations as
well as the `{{#each}}` loop, especially with syntax highlight:

```hbs
<SuperSelect @selected={{this.user.country}} as |Option|>
  {{#each this.availableCountries as |country|}}
    <Option @value={{country}}>{{country.name}}</Option>
  {{/each}}
</SuperSelect>
```

This RFC proposes that we adopt the angle bracket invocation syntax to Ember as
an alternative to the classic ("curlies") invocation syntax.

Unlike the original RFC, the angle bracket invocation syntax proposed here is
purely syntatical and does not affect the semantics. The invocation style is
largely transparent to the invokee and can be used to invoke both classic
components as well as [custom components](https://github.com/emberjs/rfcs/pull/213).

Since the original angle bracket RFC, we have worked on a few experimental
implementation of the feature, both and in Ember and Glimmer. These experiments
allowed us to attempt using the feature in real apps, and we have learned some
valuable insights throught these usage.

The original RFC proposed using the `<foo-bar ...>` syntax, which is the same
syntax used by web components (custom elements). While Ember components and web
components share a few similarities, in practice, we find that there are enough
differences that causes the overload to be quite confusing for developers.

In addition, the code needed to render Ember components is quite different
from what is needed to render web components. If they share the same syntax,
the Glimmer template compiler will not be able to differentiate between the two
at build time, thus requiring a lot of extra runtime code to support the
"fallback" scenario.

In conclusion, the ideal syntax should be similar to HTML syntax so it doesn't
feel out of place, but different enough that developers and the compiler can
easier tell that they are not just regular HTML elements at a glance.

## Detailed design

### Tag Name

The first part of the angle bracket invocation syntax is the tag name. While
web components use the "dash rule" to distinguish from regular HTML elements,
we propose to use capital letters to distinguish Ember components from regular
HTML elements and web components.

The invocation `<FooBar />` is equivalent to `{{foo-bar}}`. The tag name will
be normalized using the `dasherize` function, which is the same rules used by
existing use cases, such as service injections. This allows existing components
to be invoked by the new syntax.

Another benefit of the capital letter rule is that we can now support component
names with a single word, such as `<Button>`, `<Modal>` and `<Tab>`.

> Note: Some day, we may want to explore a file system migration to remove the
> need for the normalization rule (i.e. also use capital case in filenames).
> However, that is out-of-scope for this RFC, as it would require taking into
> consideration existing code (like services), transition paths and codemods.

### Arguments

The next part of the invocation is passing arguments to the invoked component.
We propose to use the `@` syntax for this purpose. For example, the invocation
`<FooBar @foo=... @bar=... />` is equivilant to `{{foo-bar foo=... bar=...}}`.
This matches the [named arguments syntax](https://github.com/emberjs/rfcs/pull/276)
in the component template.

If the argument value is a constant string, it can appear verbatim after the
equal sign, i.e. `<FooBar @foo="some constant string" />`. Other values should
be enclosed in curlies, i.e. `<FooBar @foo={{123}} @bar={{this.bar}} />`.
Helpers can also be used, as in `<FooBar @foo={{capitalize this.bar}} />`.

#### Reserved Names

`@args`, `@arguments` and anything that does not start with a lowercase letter
(such as `@Foo`, `@0`, `@!` etc) are reserved names and cannot be used. These
restrictions may be relaxed in the future.

#### Positional Arguments

Positional arguments (`{{foo-bar "first" "second"}}`) are not supported.

### HTML Attributes

HTML attributes can be passed to the component using the regular HTML syntax.
For example, `<FooBar class="btn btn-large" role="button" />`. HTML attributes
can be interleaved with named arguments (it does not make any difference). This
is a new feature that is not available in the classic invocation style.

These attributes can be accessed from the component template with the new
`...attributes` syntax, which is available only in element positions, e.g.
`<div ...atributes />`. Using `...attributes` in any other positions, e.g.
`<div>{{...attributes}}</div>`, would be a syntax error. It can also be used on
multiple elements in the same template. If attributes are passed but the
component template does not contain `...attributes` (i.e. the invoker passed
some attributes, but the invokee does not take them), it will be a development
mode error.

It could be thought of that the attributes in the invocation side is stored in
an internal block, and `...attributes` is the syntax for yielding to this
internal block. Since the `yield` keyword is not available in element position,
a dedicated syntax is needed.

Classic components (`Ember.Component`) will implicitly have an `...attributes`
added to the end of the wrapper element (if `tagName` is not an empty string),
after any attributes added by the component itself (using `attributeBindings`,
`classNames` etc). This means that attributes provided by the caller will
override (replace) those added by the component (except for `class`, which is
merged).

### Block

A block can be passed to the invokee using the angle bracket invocation syntax.
For example, the invocation `<FooBar>some content</FooBar>` is equivalent to
`{{#foo-bar}}some content{{/foo-bar}}`. As with the classic invocation style,
this block will be accessible using the `{{yield}}` keyword, or the `@main`
named argument per the [named blocks RFC](https://github.com/emberjs/rfcs/blob/master/text/0226-named-blocks.md).

Block params are supported as well, i.e. `<FooBar as |foo bar|>...</FooBar>`.

There is no dedicated syntax for passing an "else" block directly. If needed,
that can be passed using the named blocks syntax.

### Closing Tag

The last piece of the angle bracket invocation syntax is the closing tag, which
is mandatory. The closing tag should match the tag name portion of the opening
tag exactly. If no block is passed, the self-closing tag syntax `<FooBar />`
can also be used (in which case `{{has-block}}` will be false).

### Dynamic Invocations

In additional to the static invocation described above (where the tag name is a
statically known component name), it is also possible to use the angle bracket
invocation syntax for dynamic invocations.

The most common use case is for invoking "contextual components", as shown in
the first example:

```hbs
<SuperSelect @selected={{this.user.country}} as |Option|>
  {{#each this.availableCountries as |country|}}
    <Option @value={{country}}>{{country.name}}</Option>
  {{/each}}
</SuperSelect>
```

Because `Option` is the name of a local variable (block param), the `<Option>`
invocation will invoke the yielded value instead of looking for a component
named "option".

To avoid collision with HTML, only local variables starting with a capital
letter can be invoked this way. No other forms of expressions are allowed in
this position at this moment:

```hbs
{{!-- these do not work! --}}
<this.foo />
<@foo />

<MyForm as |f|>
  <f.input />
  <f />
</MyForm>
```

This restriction may be relaxed in the future. In the time being, the `let` can
be used to rename these into an invocable form:

```hbs
{{#let ... as |FooBar|}}
  <FooBar />
{{/let}}
```

> Note: The [named blocks RFC](https://github.com/emberjs/rfcs/blob/master/text/0226-named-blocks.md)
> proposed to use the `<@foo>...</@foo>` syntax on the invocation side to mean
> providing a block named `@foo`. This will create a potential conflict with
> this future extension. Since the named blocks RFC has not been implemented
> yet, we propose to change the block-passing syntax to `<@foo=>...</@foo>` to
> avoid any confusion and potential conflict.

## How we teach this

Over time, we will switch to teaching angle bracket invocation as the primary
invocation style for components. The HTML-like syntax should make them feel
more familiar for new developers.

Classic invocation is here to stay – the ability to accept positional arguments
and "else" blocks makes them ideal for control-flow like components such as
`{{liquid-if}}`.

## Drawbacks

Because angle bracket invocation is designed for the future in mind, allowing
angle bracket invocations on classic components might introduce some temporary
incoherence (such as the interaction between the attributes passing feature and
the "inner HTML" semantics). However, in our opinion, the upside of allowing
incremental migration outweighs the cons.

## Alternatives

We could just stick with the classic invocation syntax.
