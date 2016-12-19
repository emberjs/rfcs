- Start Date: 2016-12-19
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Allow yielding to multiple named blocks in an Ember component.

# Motivation

### Why now?

Glimmer is stable and Ember 2.x has been around for a while. It's about time to get back to this idea
since it was always a well-liked idea that just happened to never come up at the right time.

### Why ever?

Components in Ember are meant to abstract away meaningfully atomic chunks of UI behavior,
e.g. a drop down menu element (cf. [ember-power-select](https://github.com/cibernox/ember-power-select)). However, currently the interface
to the components abstraction consists of:
  * any number of attributes (and optionally many positional params)
  * a single passed block (and optionally a single passed inverse block)
You will note that you can pass many attributes or params through a component's interface,
but only a single block. I think this has led to a trend in how we design high-level, complex
components which have to be flexible to fit many use cases. We've started building components
that are more configurable than they are composable.

The goal of both configurability and composability is to provide flexibility in how you go from
input to output over a collection of interfaces in a system. Gaining this desired flexibility
via configurability tends to lead to fewer interfaces but more parameters to those interfaces.
Composability usually means many, more atomic interfaces that fit together with less parameters.
Take the example of JS functions: you can perform complex logic on input by calling a single
functions that takes in the input plus a giant config object telling you how to handle the input,
or you can break the logic into multiple functions that call each other. Logic written in a single
function with many configuration parameters will always be less flexible, maintainable, and reusable
than logic contained in many atomic functions.

In Ember components we achieve composability by passing blocks that call other components.
Configurability, on the other hand, is achieved by passing attributes and positional parameters.
Flexibility in high-level Ember UI components is determined by the variability of rendered content
that can be achieved by a single component. Limiting block passing to a single, un-named block means
we can only *wrap* the passed block in content, rather than arbitrarily compose content with multiple
passed blocks of content. This leads to things like conditionally rendering some content based on
passed attributes or parameters or conditionally yielding different block parameters. Yielding to
multiple named blocks would make the use of a lot of the configuration that's currently happening
unnecessary and in turn encourage composition of components instead.

# Detailed design

To achieve yielding to more than one named block I propose adding the following built-in syntax
and semantics to components:

> **some-component.hbs**
> ```hbs
> {{yield:block-b 'B'}}
> {{yield}}
> {{yield:block-a 'A'}}
> ```
>
> **sample-invocation.hbs**
> ```hbs
> {{#some-component}}
>   This is the default, un-named block
> {{:block-a as |block|}}
>   This is block {{block}}
> {{:block-b as |block|}}
>   This is block {{block}}
> {{/some-component}}
> ```
>
> **Result**
> ```
> This is block B
> This is the default, un-named block
> This is block A
> ```

If the default block is omitted, to ease developer ergonomics I propose the following
additional syntax:

> **some-component.hbs**
> ```hbs
> {{yield:block-b 'B'}}
> {{yield:block-a 'A'}}
> ```
>
> **sample-invocation.hbs**
> ```hbs
> {{#some-component:block-a as |block|}}
>   This is block {{block}}
> {{:block-b as |block|}}
>   This is block {{block}}
> {{/some-component}}
> ```
>
> **Result**
> ```
> This is block B
> This is block A
> ```

# How We Teach This

The proposed syntax and semantics is a logical continuation of the current Ember syntax
and semantics relating to blocks and yielding blocks. The "block" sections (`:block-b` in the example above)
of components can directly analogized to `else` sections in the current syntax for inverse blocks.
The proposed syntax can be easily taught by simply extending the current Ember guides on components,
as this is merely the addition of syntax that adds opt-in functionality.

# Drawbacks

The proposal adds more syntax to component invocation so it increases general learning curve.
The proposed syntax also might face some implementation challenges since it probably goes deep
into glimmer internals of how yields work. It also sort of duplicates the limited functionality
provided by the current `yield` helper when called with the `to` attribute for yielding to the
inverse block.

# Alternatives

### Past RFCs

This proposal was inspired by and draws knowledge from [this unresolved RFC](https://github.com/emberjs/rfcs/pull/72)
for named yields and [this closed RFC](https://github.com/emberjs/rfcs/pull/43) for multiple yields.
Ultimately there was no consensus on a syntax that was both clear and easy to teach but also technically desirable.

### Other alternatives

The main criticism of [the closed RFC]([this closed RFC](https://github.com/emberjs/rfcs/pull/43)) was that the proposed syntax was dynamic and not statically analyzable.
A dynamic version of this proposal would also be possible, looking something like this:

> **some-component.hbs**
> ```hbs
> {{yield to="block-b" 'B'}}
> {{yield to="block-a" 'A'}}
> ```
>
> **sample-invocation.hbs**
> ```hbs
> {{#some-component:block 'block-a' as |block|}}
>   This is block {{block}}
> {{block 'block-b' as |block|}}
>   This is block {{block}}
> {{/some-component}}
> ```
>
> **Result**
> ```
> This is block B
> This is block A
> ```

The dynamic alternative follows the current syntax and semantics around inverse blocks more closely, but since
yielding to inverse blocks is not even very well documented, that may not be as salient as the benefits
of static analyzability.

Another alternative is to automagically yield some sort of hash of named blocks to the primary block and then
invoke the named blocks within the main block. Something like this is described by [@runspired](https://github.com/runspired) in a [gist](https://gist.github.com/runspired/71bc9ee3a6dd0386fb23) he posted
in response to [the unresolved RFC](https://github.com/emberjs/rfcs/pull/72) for named yields.

The primary alternative for this is to just forgo named blocks completely and rely on contextual components
for composability. However, I feel there are problems with flexibility in content composability that that named
blocks/yields solve contextual components just can't. Another way to achieve [something like named yields](https://github.com/emberjs/rfcs/pull/72#issuecomment-219174876)
wihtout actually implementing it was suggested by [@foxnewsnetwork](https://github.com/foxnewsnetwork). However, the
simulation of named yields is arguably hard to teach/understand.

# Unresolved questions

Is the colon syntax readable? I tried pretty much every delimiter character and settled on colon.
What is the technical feasibility as far as changes to Glimmer that would have to be made?
As [@mmun](https://github.com/mmun) mentioned in [the closed RFC](https://github.com/emberjs/rfcs/pull/43) for multiple yields, could this be better
implemented as a handlebars language feature that implements passing blocks around, foregoing semantic/syntactic
interactions with yield altogether? What are the pros and cons to that route?
