- Start Date: 2018-03-23
- RFC PR: https://github.com/emberjs/rfcs/pull/317
- Ember Issue: (leave this empty)

# Named Block Syntax

## Summary

This RFC finalizes the syntax for named blocks. There are two changes relative to the [original RFC](https://github.com/emberjs/rfcs/blob/master/text/0226-named-blocks.md):

- An equal sign (`=`) is now required after the block name
- Block names are restricted to `@[a-z][A-Za-z0-9-]*`

Here's an example with curly invocation style

```
{{#power-accordian items=blogPosts}}
  <@header= as |item|>
    {{item.title}}
  </@header>

  <@details= as |item|>
    <h4>By {{item.author}}</h4>
    <p>{{item.text}}</p>
  </@details>
{{/power-accordian}}
```

and with angle bracket invocation style

```
<PowerAccordian @items={{blogPosts}}>
  <@header= as |item|>
    {{item.title}}
  </@header>

  <@details= as |item|>
    <h4>By {{item.author}}</h4>
    <p>{{item.text}}</p>
  </@details>
</PowerAccordian>
```

## Motivation

Requiring an equal sign makes it clearer that named blocks are arguments to the component. Limiting the valid block names ensures that block names are easy to parse with a computer and easy to read by a human.

## Detailed design

The grammar for named block start and end tags is

```
ws = \s+                    // whitespace
id = [a-z][A-Za-z0-9-]*     // e.g. foo-bar, fooBar, foo2, ...

AtIdentifier        = '@' id
NamedBlockStartTag  = '<' ws? AtIdentifier ws? '=' ws? BlockParams? ws? '>'
NamedBlockEndTag    = '</' ws? AtIdentifier ws? '>'
```

where `BlockParams` is defined in the [Handlebars grammar](https://github.com/wycats/handlebars.js/blob/1e954ddf3c3ec6d2318e1fadc5e03aaf065b2fbd/src/handlebars.yy#L136-L138).

It is also required that the start and end identifiers must be equal, like in HTML.

#### Examples of valid start tags

```
<@header= as |item|>
< @header = as |item|  >
< @header =>
```

#### Examples of invalid start tags

```
<@ header=>
<@header>
<@header as |item|>
```

#### Examples of valid end tags

```
</@header>
</ @header >
```

#### Examples of invalid end tags

```
</header>
< /@header>
```

## How we teach this

All the considerations from the original RFC apply, but now it will be easier to see that named blocks are arguments.

## Drawbacks

There are some drawbacks to this proposal:

#### The equal sign may trip up syntax highlighters

This is likely true but it's also likely true of `@` symbol as well. Regardless, we should encourage the community to update their syntax packages to work with named blocks. We probably want the named block "tag" name to syntax highlight the same way as the arg in `<Foo @bar=123>`.

#### The equal sign in that position doesn't feel like HTML

This is by design. Named blocks are intentionally not supposed to look like HTML so that they are not accidentally interpreted to be the content of a single-block style invocation. Instead they are designed to look like args, because they are args!

## Alternatives

We could leave the named block syntax the way it is but we would lose the important link between named blocks and args.
