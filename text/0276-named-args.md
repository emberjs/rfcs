---
Start Date: 2017-12-10
RFC PR: https://github.com/emberjs/rfcs/pull/276
Ember Issue: https://github.com/emberjs/ember.js/pull/15968

---

# Summary

Introduce `{{@foo}}` in as a dedicated syntax for a component's template to
refer to named arguments passed in by the caller.

For example, given the invocation `{{hello-world name="Godfrey"}}` and this
component template in `app/templates/components/hello-world.hbs`:

```hbs
Hello, {{@name}}
```

Ember will render "Hello, Godfrey".

# Motivation

Currently, the way to access named arguments passed in from the caller is to
reference `{{name}}` in the template. This works because when Ember creates
the component instance, it automatically [assigns](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign)
all named arguments as properties on the component instance.

The first problem with this approach is that the `{{name}}` syntax is highly
ambigious, as it could be referring to a local variable (block param), a
helper or a named argument from the caller (which actually works by accessing
auto-reflected `{{this.name}}`) or a property on the component class (such as
a computed property).

This can often lead to confusion for readers of the template. Upon encountering
`{{foo}}` in a component's template, the reader has to check all of
these places: first you need to scan the surrounding lines for block
params with that name; next you check in the helpers folder to see if there
is a helper with that name (it could also be coming from an addon!); then you
check if it is an argument provided by the caller; finally, you check the
component's JavaScript class to look for a (computed) property. If you _still_
did not find it, maybe it is a named arguments that is passed only sometimes,
or perhaps it is just a leftover reference from a previous refactor?

Providing a dedicated syntax for referring to named arguments will resolve the
ambiguity and greatly improve clarity, especially in big projects with a lot
of files (and uses a lot of addons). (The existing `{{this.name}}` syntax can
already be used to disambiguate component properties from helpers.)

As an aside, the ambiguity that causes confusion for human readers is also a
problem for the compiler. While it is not the main goal of this proposal,
resolving this ambiguity also helps the rendering system. Currently, the
"runtime" template compiler has to perform a helper lookup for every `{{name}}`
in each template. It will be able to skip this resolution process and perform
other optimizations (such as reusing the internal [reference](https://github.com/glimmerjs/glimmer-vm/blob/master/guides/04-references.md)
object and caches) with this addition.

Another problem with the current approach of automatically "reflecting" named
arguments on the instance is that they can unexpectedly overwrite other
properties defined on the component's class. It also defeats performance
optimizations in JavaScript engines as this approach creates many different
polymorphic "shapes" for instances that otherwise belong to the same
component class.

While this proposal does not directly solve this problem (we are not proposing
that we deprecate or remove the "auto-reflection" on `Ember.Component`), it
paves the way for a future world where components can work without them.

Notably, the current iteration of the [Glimmer Components](https://glimmerjs.com/guides/templates-and-helpers)
have adopted this design for over a year now and the experience has been very
positive. This would be one of the first pieces (admittedly, only a tiny piece)
of the Glimmer.js experiment to make its way into Ember. We think this feature
is small, self-contained but useful enough to be the ideal candidate to kick
off this process.

# Detailed design

This feature was baked into the Glimmer VM very early on. In fact, the
only thing that is stopping them from working in Ember is [an AST transform](https://github.com/emberjs/ember.js/blob/87be17d8e69f83b2abed8c0695f8fa5e4bcae473/packages/ember-template-compiler/lib/plugins/assert-reserved-named-arguments.js)
that specifically disallows them. Therefore, "implementing" this feature is
just a matter of deleting that file.

Additionally, the legacy `{{attrs.foo}}` syntax (which more or less tries to
accomplish the same thing) has actually been [implemented using `{{@foo}}`](https://github.com/emberjs/ember.js/blob/87be17d8e69f83b2abed8c0695f8fa5e4bcae473/packages/ember-template-compiler/lib/plugins/transform-attrs-into-args.js)
under-the-hood since Ember 2.10.

## Reserved Names

We will reserve `{{@args}}`, `{{@arguments}}` and anything that does not
start with a lowercase letter (such as `@Foo`, `@0`, `@!` etc) in the first
version. This is purely speculative and the goal is to carve out some space
for future features. If we don't end up needing them, we can always relax
the restrictions down the road.

# How We Teach This

`{{@foo}}` is the way to access the named arguments passed from the caller.

Since the `{{foo}}` syntax still works on `Ember.Component` (which is the
only kind of components available today) via the auto-reflection mechanism,
we are not really in a rush to migrate the community (and the guides, etc)
to using the new syntax. In the meantime, this could be viewed as a tool to
improve clarity in templates, similar to how the optional "explicit `this`"
syntax (`{{this.foo}}`).

While we think writing `{{@foo}}` would be a best practice for new code
going forward, the community can migrate at its own pace one component at a
time.

We can also encourage the community to supplement this effort by wiring
linting tools and code mods.

# Drawbacks

This introduces a new piece of syntax that one would need to learn in order to
understand Ember templates.

This mostly affects "casual" readers (as this should be very easy for an Ember
developer to learn, understand and remember after encounting/learning it for
the first time). However, since these casual readers are also among those
who are most acutely affected by the ambiguity, we believe this is still a
net improvement over the status-quo.

# Alternatives

We have `{{attrs.foo}}` today. In React, there is `this.props.foo`.

Given how common this is, we think it deserves its own dedicated, succinct
syntax. The other alternatives that involve reflecting them on the component
instances also would not allow for the internal optimizations in the Glimmer
VM.
