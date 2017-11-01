- Start Date: 2017-10-27
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Introduce a new form of `{{partial}}` without the "context-shifting"
behavior (automatically getting access to any variables available to the
calling context) and deprecate the current form.

```hbs
{{#each model as |post|}}
  {{#with post.canonicalUrl as |url|}}
    {{!-- The usual form of partial will automatically inherit all the
    variables in scope (such as `post` here) plus everything else on
    `this` (such as `model`) --}}
    {{partial "post-header"}}

    {{#each post.authors as |author|}}
      {{!-- This partial will only have access to the explicitly passed
      `post` and `author` (via the name `contributor`) variables, but not
      `url`, `model` and everything else on `this` --}}
      {{partial "contributor-card" post=post contributor=author}}
    {{/each}}
  {{/with}}
{{/each}}
```

# Motivation

In its current form, `{{partial}}` is the only remaining "context-shifting
helper" from the pre-1.0 era. That is, the "context" (what variables are
available to the partial) depends on the calling context. This makes partials
incredibly difficult to understand and is a refactoring hazard.

For example, you might scan at a template and decide a variable (either
defined on the controller/component, or introduced in the template via a
block param) is unused and can be safely removed. However, that assumption
might not hold if there are one or more `{{partial}}` invocations in the
template.

This is contrary to components – where their invocation requires you to pass
in any data explicitly, in the form of arguments. Not only does this make it
possible to freely refactor the calling template (as long as you don't change
or remove the passed arguments, the component should continue to work), it
also offers some hints about what the component might do, as opposed to partials
where its name would be the only clue.

Drawing parallel to JavaScript code, if component invocations were function
calls, then partial invocations are `eval`:

```js
class MyTemplate {
  // ...
  render() {
    let { model, name } = this;

    MyForm(f => {
      // Invoking a component is similar to calling a function. You can
      // directly see what data are being passed and feel confident about
      // any refactor that does not change said data (such as replacing
      // `model: this.model` with just `model` and renaming the outer
      // `name` variable to make clear it is unrelated to the `name` being
      // passed here.
      AnotherComponent({ f, model: this.model, name: "Godfrey" });

      // Invoking a partial, on the other hand, is similar to an `eval`.
      // You cannot tell what variables are being used inside the without
      // carefully reading its source, so any refactor here (such as
      // removing the unused `model` and `name` destructure at the top)
      // will likely break the code.
      eval(AnotherPartial);
    });
  }
}
```

Not only does this make partials difficult for humans to understand, all of
these problems apply to machines as well. In fact, the "function" vs "eval"
analogy is more than just a illustration of how to reason about their difference,
it actually closely matches the way these two features are modelled/implemented
in the Glimmer VM. This makes it really complicated and [really][partial-bug-1]
[error][partial-bug-2]-[prone][partial-bug-3] to implement `{{partial}}`.

[partial-bug-1]: https://github.com/glimmerjs/glimmer-vm/pull/707
[partial-bug-2]: https://github.com/glimmerjs/glimmer-vm/pull/730
[partial-bug-3]: https://github.com/emberjs/ember.js/pull/15777

Futhermore, the extra book keeping to enable what amounts to an `eval()` also
means they are much slower than regular components. (The opposite used to be true
so users who were used to using partials as a performance escape-hatch are
probably not getting what they expect.)

Due to all these problems with "context-shifting partials", for a long time the
core team has considered `{{partial}}` (in its current form) a legacy feature
and wanted to discourage its use in new apps. However, our deprecation policy
requires us to provide a transition path before deprecating any features, and
since all available alternatives today (see the "Alternatives" section) have
significant drawbacks, we were not able to deprecate this feature and communicate
our preference to the community.

By introducing a non-context shifting form of `{{partial}}` that does not have
any of these drawbacks, we will be able to signal our intent to remove the
feature and transition the community to a better alternative immediately.

# Detailed design

Currently, the `{{partial}}` helper only take one positional arguments (the name
of the partial). We will use the presence of any named arguments as an opt-in for
the new semantics.

Specifically:

1. `{{partial "foo"}}` will invoke the `foo` partial template with the current
  `this` as its context and make available any local variables that are in-scope
  on the invocation side.

2. `{{partial "foo" bar="baz"}}` will invoke the `foo` partial template with `this`
  set to `{ bar: "baz" }` (a POJO consisting of only the named arguments passed to
  the partial), and does not make any outer local variables available to the
  template.

Users who are familiar with how component invocations work today should find the
second form quite familiar and easy to understand (which further highlights the
oddities of the first/current form).

In addition, we will pair this change with a deprecation. When a partial template
is invoked in the first form **and** it attempts to access any variable on `this`
or on the outside scope (i.e. any `{{foo}}` that does not resolve into a helper),
we will issue a deprecation warning suggesting the user to switch to the new
invocation style.

While the named arguments can be used as an opt-in for cases where external data
are needed, the `{{partial "foo"}}` invocation is actually ambigious. In this
case, there are three possibilities:

1. This is an existing "old-style" ("context-shifting") partial where the "foo"
   template depends on access to the outside scope.

2. This is an existing "old-style" ("context-shifting") partial where the "foo"
   template does not depend on access to the outside scope (e.g. a site footer).

3. This is a "new-style" ("non-context-shifting") partial where the "foo"
   template _happens_ to not take any arguments (e.g. a site footer).

In the first case, there will be a deprecation warning nudging the user to refactor
into the new-style invocation with named arguments.

On the other hand, there are no ways to differentiate between the second and the
third case. *However, there are no observable differences between the two.* Notably,
there will be no deprecation (since the warning is triggered by actually accessing
the outer scope, not merely from invoking an old-style partial). Since there are no
external data-dependency, when Ember drops support for "context-shifting" partials,
the code will continue to work seamlessly even if the underlying implementation
strategy changes.

Speaking of internal implementation strategy, the new-style partial as described in
this proposal can be implemented completely in "user-land" (as opposed to inside
Glimmer VM) with a combination of AST-transform and using Glimmer's component system:

1. Introduce an AST-transform that turns any new-style partial invocation from
   `{{partial name foo=bar ...}}` into `{{-internal-new-partial name foo=bar ...}}`.

2. Implement `{{-internal-new-partial}}` as a new kind of light-weight component
   using the component manager API. This light-weight component will simply capture
   any named arguments and return it as the "self" reference and does not have any
   template wrapper or lifecycle-hooks.

The only exception is the deprecation message. Since the actual triage for `{{foo}}`
happens [deep inside the VM internals][resolve-maybe-local], we would probably have
to expose additional hooks (or other mechanisms) for Ember to issue the deprecation
warning.

[resolve-maybe-local]: https://github.com/glimmerjs/glimmer-vm/blob/f78e91b47af9195d2ab063cb24e12ae5f1e5c9be/packages/%40glimmer/opcode-compiler/lib/syntax.ts#L272

# How We Teach This

For new users, we can teach the new-style `{{partial}}` as if it is just a light-weight
(template-only) component that does not have/need any behavior associated with it.
Since they are actually implemented as components under-the-hood, it will be a completely
leak-free analogy.

Because of that, it might even be a good training wheel for teaching components in Ember
to a completely new user: start with a big template, and when it grows *too* big, you can
extract it into a partial. Once you have done that, since you have already identified and
passed in any external data-dependency, you will be ready to "upgrade" to a component
whenever you need to add behavior (or adding CPs, etc) by (more-or-less) just adding a
JavaScript file and some minor changes to the invocation.

It is also probably the case that for someone who is completely new to Ember, they would
have no reason to expect partials to work the way they do today – after all, it is quite
normal to have to explicitly pass in any data you need in every other context (component
invocation, function calls, etc). So, arguably, this is actually one *fewer* thing to
teach and might actually align better with everyone's mental model of how programming
works.

For existing users who are already familiar with how components work, this should also
be quite easy to teach. Admittedly, the challenge (or potential confusion) arises from
having to "forget" how partials work today. This should be addressable with a clear
deprecation message.

Also, as we have been promoting Ember's switch away from context-shifting helpers for
quite a while now, arguably it also just amounts to "forgetting a special exception to
the general rule", which should make this feels quite nice and coherent in the long run.

Once you have completed the migration (following the deprecation warning), it will also
makes it much easier to refactor partials into components if/when there is a need for
them.

# Drawbacks

* "How is it different from a component?"

  The new-style component is, in fact, a component. The point of this proposal is to
  (over time) eliminate a special construct (partials) and unify them with an existing
  one (components).

  It might have been nice if we could just remove it and replace it directly with Just
  Components™, but we are not quite there yet (see the "Alternatives" section).

* `{{partial "foo"}}` is ambigious

  As mentioned above, it is not possible to express "new-style partials but without any
  arguments" due to the choice of the opt-in syntax. However, also as mentioned above,
  there are no observable differences in semantics. It does mean that the "content-only
  partials" will be a little slower than they need to be (since we must use the old
  implementation due to the possible ambiguity), but this is unlikely to matter in
  practice.

* Creating extra churn when adding Glimmer components

  When "Glimmer components" land, the "new-style partial" functionality will be rendered
  obsolete as "template-only Glimmer components" will offer a superset of its features
  and be just as easy to use. Presumably, when that happens, the partial feature will be
  phased out and removed from core. Therefore, landing this feature now might introduce
  extra churn.

  However, once our users have migrated to "new-style partials", it will be trivial to
  upgrade them into Glimmer components with an automatic code mod (assuming [Module
  Unification][module-unification]). The hard part in the transition is to figure out
  the implicit data dependency from the "context-shifting partials", which needs to
  happen no matter what. Therefore, in aggregrate there is very little wasted effort
  and this feature will serve as an important intermediate step in the transition.

  Furthermore, since "new-style partials" does not have any of the downsides of
  "context-shifting partials", it becomes much less urgent to deprecate and remove
  the feature, as the only motivation would be to reduce a redundant concept (as
  opposed to discourage a hazardous programming model). Therefore, we can choose to
  phase out the feature more slowly ("soft-deprecation").

  Finally, since the feature is implementable in user-land (again, as opposed to
  inside Glimmer VM), we can support this feature for the foreseeable future in a
  supported addon separate from core when the time comes. (In fact, with the
  [Custom Components API][custom-components] it will be possible to implement this
  purely using Ember's public API alone.)

  [module-unification]: https://github.com/emberjs/rfcs/blob/master/text/0143-module-unification.md
  [custom-components]: https://github.com/emberjs/rfcs/pull/213

# Alternatives

The ultimate goal of this proposal is to deprecate "context-shifting partials" as
they exist today. The "new-style partials" propose exists merely to provide a
transition path to satisfy the requirements to deprecate the feature. Here are some
alternative transition paths:

* Curly components

  If the goal is just to replace partials with components, we could simply advise
  our users to "use components instead". In today's world, that would mean replacing
  them with curly components (`Ember.Component`).

  The problem with curly components is that they will introduce a wrapper element by
  default, which might break the page's layout/CSS. To work around that, they would
  have to either add a JavaScript class for the component to write `tagName: ''` or
  to supply that on the invocation side.

  The current consensus is that this is "too much of a lift" that makes an otherwise
  very simple usecase feels "too heavy", especially when teaching partials as a simple
  refactoring tool to new users.

* Glimmer components

  Another option is to do nothing today and wait for Glimmer components to land, which
  is the status quo. In my opinion, this has been the status quo for too long. It is
  quite unfortunate that we were not able to actively discourage new partials from
  being written when the majority of the core team and a large part of the community
  are already aware of the hazards. In general, we should also decouple feature
  dependencies as much as possible.

  As mentioned above, it should be possible to automatically migrate all "new-style
  partials" to "template-only Glimmer components" with a codemod once they land (whereas
  migrating "context-shifting partials" seems much more difficult).

  That being said, I definitely would like more eyes/brains on this claim to make sure
  it is in fact doable, and where possible, we should (speculatively?) align the proposed
  semantics with "template-only Glimmer components" to make our job easier in the future.

  For the codemod to work, there are two sides to the equation – the invocation sites
  and the partial templates.

  On the invocation side, as long as they all go through the `{{partial ...}}` keyword
  (which would argue for disallowing invoking a new-style partial with the component
  helper), they should be trivial to discover and rewrite.

  On the template side, as far as I am aware, the only deviation from "template-only
  Glimmer components" is instead of assigning and accessing the passed arguments via
  `this`, they will likely be accessed through the `@foo` syntax (which is similar to
  `attrs.foo` in curly components today).

  In theory, assuming the project has also adopted [Module Unification][module-unification],
  it should be easy to identify all of the known helpers and partial templates in a
  project as they will be located in their own collection. Once we have that information,
  it should just be a matter of resolving each `{{foo}}` in all partial templates
  statically and rewrite anything that is not a helper into `{{@foo}}`.

  However, for historical reasons, the internal partial template lookup rules are
  currently quite "liberal", so identifying all partials templates of a project
  might turn out to be challenging or even impossible in practice. (It is unclear
  at this point how much of those will be carried over to the Module Unification
  world.)

  Therefore, if possible, it is better to align the template-side semantics perfectly
  so that the codemod would only need to worry about the invocations. One possibility
  is to "fast-track" a proposal that simply enables the `@foo` syntax before landing
  this proposal. This would allow us to drop the `this` POJO and allow for a seamless
  upgrade to "template-only Glimmer components" (based on what we know abou them
  today, which is subject to change of course).

  [module-unification]: https://github.com/emberjs/rfcs/blob/master/text/0143-module-unification.md

* Disallow variables in partials

  Another option is to deprecate accessing *any* variables inside partials. This only
  works for simple "content-only" partials (e.g. a site footer). However, there are
  a lot of existing use cases for partials that does not fit that description.

In addition, we could also...

* Deprecate partials without providing an alternative

  This would violate our deprecation policy.

* Keep the current `{{partial}}` indefinitely

  I already enumerated the problems with the features, but I would be curious to know
  if anyone is interested in representing/advocating for this.

# Unresolved questions

* Should we drop the prefix `-` (in the template filename) requirement?
* Should we move partials into the components folder?
* What should happen if you do `{{partial-name ...}}`?
