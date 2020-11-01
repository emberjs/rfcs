---
Start Date: 2019-03-06
Relevant Team(s): Ember.js
RFC PR: #460
Tracking: (leave this empty)

---

# Yieldable Named Blocks

## Summary

This RFC amends [#226 (named blocks)][named-blocks] to finalize the syntax of named blocks and reduce the scope of the feature.

[named-blocks]: https://emberjs.github.io/rfcs/0226-named-blocks.html

## Motivation

The original named blocks RFC (#226) defined a mechanism for passing multiple blocks to a component, as well as semantics for capturing the block in JavaScript.

Since #226, RFC [#311 (angle bracket invocation)][angle-bracket] repurposed the original syntax for invocation.

```hbs
<Tab>
  <@header></@header>
</Tab>
```

In #226, the above syntax passed a named block called `@header` to the `Tab` component. RFC #311 repurposed that syntax to mean "invoke the component located at `@header` in the default block of `Tab`".

Curly bracket components already support two blocks (the default block and the `else` block), while angle bracket components, as implemented today in Ember, only support a single default block.

While RFC #226 predates a formal RFC about angle bracket components, it speculates significantly about usage in angle bracket components, and is heavily motivated by the desire to support more than a single block in that context.

That said, #226 included significant additional scope beyond allowing angle bracket invocations to pass more than one block, including:

- using `<@name>` as a syntax for passing named blocks to curly components
- passing named blocks as `@name` variables, accessible in the receiving component as `@blockName`
- capturing named blocks in JavaScript using `this.blockName`, as well as capturing named blocks in JavaScript by forwarding them into JavaScript as a parameter to a helper or another named argument.

While versions of those features are all desirable, the RFC, in total, had a fairly large scope and also speculated a lot about the future of angle bracket components which would not be approved until RFC #311.

This RFC proposes a version of named blocks with a smaller scope, with a plan to reintroduce the other goals of the original named blocks RFC in a subsequent RFC.

> Note: The version of named blocks described in this RFC has already been implemented in Glimmer VM.

[angle-bracket]: https://emberjs.github.io/rfcs/0311-angle-bracket-invocation.html

## Detailed design

> The detailed design of this RFC assumes a starting point without RFC #226, and describes the functionality in terms of RFC #311.

When invoking a component with angle bracket syntax, the content between the starting tag and the ending tag is passed as a block to the receiving component.

This means that the receiving component can invoke the block using `{{yield}}`. The block is not exposed to JavaScript at all, and it may only be invoked in the body of the receiving component's template.

On their own, blocks provide useful functionality. For example, consider an `Article` component that takes a `@body` argument:

```hbs
<Article @body={{this.body}} />
```

The `Article` component could be implemented this way:

```hbs
<article>
  <section>{{@body}}</section>
</article>
```

This works, but what if you want to include HTML functionality or use helpers. In that case, you could use a block:

```hbs
<Article>
  <div class='byline'>{{byline this.author}}</div>
  <div class='body'>{{this.body}}</div>
</Article>
```

The `Article` component could then be implemented by using `{{yield}}` to invoke the block:

```hbs
<article>
  <section>{{yield}}</section>
</article>
```

But what if we also want the ability to pass a title to the component. We could use a `@title` argument:

```hbs
<Article @title={{this.title}}>
  <div class='byline'>{{byline this.author}}</div>
  <div class='body'>{{this.body}}</div>
</Article>
```

Implemented this way:

```hbs
<article>
  <header>{{@title}}</header>
  <section>{{yield}}</section>
</article>
```

Again, that works, but if you want to use any of the block features, such as the ability to use HTML, you can't pass a second block.

This RFC introduces the ability to pass any number of blocks to a component. With that capability, you could call `Article` this way:

```hbs
<Article>
  <:title>
    <h1>{{this.title}}</h1>
  </:title>
  <:body>
    <div class='byline'>{{byline this.author}}</div>
    <div class='body'>{{this.body}}</div>
  </:body>
</Article>
```

And implement it this way:

```hbs
<article>
  <header>{{yield to='title'}}</header>
  <section>{{yield to='body'}}</section>
</article>
```

### Block Parameters

Just like normal blocks, named blocks can take block parameter, which are passed via `yield`.

```hbs
<Article @article={{this.article}}>
  <:header as |title|>
    <h1>{{title}}</h1>
  </:header>
  <:body as |body|>
    <div>{{body}}</div>
  </:body>
</Article>
```

```hbs
<article>
  <header>{{yield @article.title to='header'}}</header>
  <section>{{yield @article.body to='body'}}</section>
</article>
```

To summarize, today's Ember allows all components to pass a default block, and curly components to pass a default block and an `inverse` block. Component templates can invoke default templates by using `{{yield}}` and `else` templates by using `{{yield to='inverse'}}`.

This RFC introduces a new syntax for passing arbitrary blocks to angle bracket components that can be invoked by the receiving component using the same `yield` syntax.

### Syntax

This RFC proposes an extension to the angle bracket invocation syntax.

From:

```
AngleBracketInvocation :
  | AngleBracketWithBlock
  | AngleBracketWithoutBlock

AngleBracketWithBlock :
  "<" ComponentTag ComponentArgs? BlockParams? ">"
  Block
  "</" ComponentTag ">"

AngleBracketWithoutBlock :
  "<" ComponentTag ComponentArgs? "/>"
```

To:

```
AngleBracketInvocation :
  | AngleBracketWithBlocks
  | AngleBracketWithBlock
  | AngleBracketWithoutBlock

AngleBracketWithBlock :
  "<" ComponentTag ComponentArgs? BlockParams? ">"
  BlockBody
  "</" ComponentTag ">"

AngleBracketWithBlocks :
  "<" ComponentTag ComponentArgs? BlockParams? ">"
  NamedBlock+
  "</" ComponentTag ">"

NamedBlock :
  | "<:" Identifier "/>"
  | "<:" Identifier BlockParams? ">" BlockBody "</:" Identifier ">"

AngleBracketWithoutBlock :
  "<" ComponentTag ComponentArgs? "/>"

ComponentTag :
  | Identifier ; when ident is in scope variable
  | Path
  | AtName
```

A component invocation has named blocks if the first non-whitespace token after the invocation's closing `">"` is `"<:"`. In that case, the whitespace around the named blocks is ignored. Otherwise, the contents between `">"` and `"</"` are the default block.

This RFC does not propose an extension to curly syntax, although a future extension to curly syntax is expected.

### Semantics

A named block in angle bracket invocation has identical semantics to the `inverse` block created by `{{else}}`. It is invoked in the receiving component by `{{yield to='<Identifier>'}}`. Block parameters may appear after `<:identifier` as `<:identifier as |block params|>`, and they are passed by the receiving component through positional parameters to `{{yield}}`.

In all observable ways, named blocks passed by angle bracket components behave like the `inverse` block passed by curly components.

This includes `has-block` and `has-block-params`, which take the name of the block as a parameter.

This RFC intentionally does not reify the block as an argument, either as an `@name` or into JavaScript. A future RFC is expected to specify and describe this kind of capturing semantics.

[glimmer-component]: https://emberjs.github.io/rfcs/0416-glimmer-components.html

## How we teach this

This feature could be taught by example, using a situation where multiple blocks are relevant.

For example, the guides could describe an accordion widget, with two named blocks:

```hbs
<Article>
  <:title>This article is awesome!</:title>

  <:body>
    My blog has very awesome content, and everyone should
    read it.
  </:body>
</Article>
```

It should be taught after `yield`, so that describing how to implement a component that uses named blocks can lean on the existing knowledge of `yield`.

```hbs
<div class="accordion">
  <h1>{{yield to='title'}}</h1>
  <div class="body">{{yield to='body'}}</div>
</div>
```

## Drawbacks

This feature introduces the concept of arbitrary named blocks, which adds cognitive overhead to the component model. On the other hand, it leans on the existing concept of blocks, including the existing well-trod syntax for block parameters and yielding.

Unlike the original #226, this RFC chooses not to provide a counterpart to this feature in curly components. RFC #226 used angle-bracket syntax inside of curly components for specifying named blocks:

```hbs
{{#await this.article}}
  <@loaded as |article|>
    <article>
      <header><h1>{{article.title}}</h1></header>
      <section>{{article.body}}</section>
    </article>
  </@loaded>
  <@error as |reason|>
    <p class='error'>Couldn't load article: {{reason}}</p>
  </@error>
  <@loading>
    <img src="/assets/loading.gif">
  </@loading>
{{/await}}
```

This was always a strange syntax, and this RFC prefers to consider alternatives once named blocks in angle bracket syntax has been absorbed.

Additionally, it might make sense to support named blocks together with `else`, and defining the semantics for that behavior requires additional design.

One possible design contemplated by the authors of this RFC would look something like:

```hbs
{{#await this.article}}
{{when :loaded}}
  <article>
    <header><h1>{{article.title}}</h1></header>
    <section>{{article.body}}</section>
  </article>
{{when :error}}
  <p class='error'>Couldn't load article: {{reason}}</p>
{{else}}
  <img src="/assets/loading.gif">
{{/await}}
```

In this design, the `else` block would be rendered if none of the other named blocks were rendered, which is similar to the semantics of `{{#each}}`, which invokes the `else` block if the block passed to `#each` is never invoked (because the iterator had no values).

This design could make sense if curly invocation ends up being useful in the long-term for control flow (equivalent to `if`, `let` and `each`). The idea would be that in curly invocation, only one of the cases is expected to be invoked at a time, and if none of the cases are invoked, the `else` block is invoked.

This is just one possible design for curlies, but it is a possible design that the RFC authors believe should be explored separately.

## Alternatives

This RFC chooses not to specify "block capturing", which is useful for features like `ember-elsewhere`. It's a genuinely useful feature that we expect to be specified in the future. However, that design is orthogonal to the use-cases anticipated by yieldable named blocks.

We could wait for a complete design on either (a) named blocks in curly components or (b) capturable named blocks. However, we believe that there's enough standalone value in this RFC to justify shipping it before finalizing either of those two designs.
