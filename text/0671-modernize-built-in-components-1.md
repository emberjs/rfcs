- Start Date: 2020-10-02
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/issues/671
- Tracking: (leave this empty)

# Stop Leaking Implementation Details of Built-in Components

## Summary

In order to stop leaking implementation details of built-in components, we
propose to:

1. Deprecate importing the following modules
   1. `@ember/component/checkbox`
   2. `@ember/component/text-area`
   3. `@ember/component/text-field`
   4. `@ember/routing/link-component`
2. Deprecate accessing the following properties on the `Ember` global
   1. `Ember.Checkbox`
   2. `Ember.LinkComponent`
   3. `Ember.TextArea`
   4. `Ember.TextField`
   5. `Ember.TextSupport` (already private, no import path available)
3. Deprecate calling `reopen` or `reopenClass` on the classes and mixins listed
   above, whether they were obtained through an import or the `Ember` global
4. Deprecate calling `reopen` or `reopenClass` on the `Ember.Component` super
   class (which is also the default export of `@ember/component`), but not when
   called on a subclass of `Ember.Component` other than those listed above
5. Deprecate calling `lookup` or `factoryFor` on an `Owner` with the following
   specifiers
   1. `component:input`
   2. `component:link-to`
   3. `component:textarea`
   4. `component:-checkbox` (already considered private)
   5. `component:-text-field` (already considered private)
   6. `template:component/input`
   7. `template:component/link-to`
   8. `template:component/textarea`
   9. `template:component/-checkbox` (already considered private)
   10. `template:component/-text-field` (already considered private)
6. Deprecate overriding the following factories on an `Owner`, through placing
   files the corresponding locations in the `app` tree or through any other
   means such as runtime registrations or using a custom `Resolver`
   1. `component:-checkbox`
   2. `component:-text-field`
   3. `template:component/input`
   4. `template:component/link-to`
   5. `template:component/textarea`
   6. `template:component/-checkbox` (already considered private)
   7. `template:component/-text-field` (already considered private)

## Motivation

The ultimate goal is to stop leaking implementation details of built-in
components and be able to move away from subclassing `Ember.Component` in their
internal implementations.

This RFC is part of a bigger plan to accomplish this goal, which requires at
least one other follow-up RFC. This section will attempt to provide context for
this overall plan. Please note that this RFC does not attempt to address every
problem mentioned here and should ultimately be evaluated on its own merits.

### The Problems

Ember's built-in components, that is `<Input>`, `<Textarea>` and `<LinkTo>`,
are currently implemented as subclasses of `Ember.Component`, also known as the
"classic" component API. This made sense historically, as that was the only
component API available in Ember at the time. While this is arguably an
implementation detail, in practice, this aspect of their implementations has
leaked and become relied upon.

Since then, we have identified some design flaws of the `Ember.Component` API,
which motivated the [Custom Components](0213-custom-components.md) and
[Glimmer Components](0416-glimmer-components.md) API, the latter of which
became the primary component programming model in modern Ember as of the
[Octane Edition](0364-roadmap-2018.md).

The same design flaws of `Ember.Component` that impacted Ember app developers
have also had an impact on these built-in components, and we would like to move
away from their legacy implementation for largely the same reason as Ember app
developers.

* * *

The most concerning aspect of these design flaws was that any named arguments
passed to a classic component during invocation will be set as properties on
the component instance. For example:

```hbs
{{!-- This is NOT supported, do not do this! --}}
<Input
  @init={{this.customInit}}
  @willDestroyElement={{this.customWillDestroyElement}}
/>
```

Here, the `@init` and `@willDestroyElement` arguments will be made available as
`instance.init` and `instance.willDestroyElement`, essentially replacing and
overriding the `init` and `willDestroyElement` methods on the component class,
which are part of the `Ember.Object` and `Ember.Component` API, respectively.

Clearly, this is not something that we intended to support, but it is simply a
consequence that falls out of the design of the `Ember.Component` API design.

A more likely example of this is the event handling hooks that we inherited
from `Ember.Component`. In classic components, the way to handle events on the
component's element is to implement hooks like `click()` and `keyDown()` on the
component class, which is what the implementation of the component does.

Even though these hooks considered private implementation details (and [clearly
marked as such](https://github.com/emberjs/ember.js/blob/74a99a2971d327783b23a1093cf9564265db1a9d/packages/%40ember/-internals/views/lib/mixins/text_support.js#L299)),
developers have observed that they can pass callbacks into arguments of the
same name during invocation, and they will be called when the corresponding
events are dispatched to the components. For example:

```hbs
{{!-- This is NOT supported, do not do this! --}}
<Input
  @click={{this.myClickCallback}}
  @focusIn={{this.myFocusInCallback}}
  @keyDown={{this.myKeyDownCallback}}
/>
```

This is essentially the same category of bugs as passing `@init` as an
argument, but it has become relatively common practice and even made it into
the official guides at one point (this is currently being addressed).

* * *

On top of this, until [Angle Bracket Invocations](0311-angle-bracket-invocation.md)
were introduced, there was no convenient way of passing HTML attributes to a
component. Components implemented using the `Ember.Component` API relied on the
`attributeBindings` API to map instance properties to HTML attributes.

The built-in components were historically bound by the same constraints, so
they are documented to take [a large amount of HTML attributes as arguments](https://github.com/emberjs/ember.js/blob/74a99a2971d327783b23a1093cf9564265db1a9d/packages/%40ember/-internals/glimmer/lib/components/text-field.ts#L62-L122).

Post-Octane, this has caused some confusion as to what should be passed as
arguments and what should be passed as HTML attributes using the angle bracket
invocation syntax. With a few rare exceptions, the latter should suffice and
the named arguments are mostly obsolete at this point.

* * *

Similarly, until [modifiers](0373-Element-Modifier-Managers.md) were available
[with Angle Bracket Invocations](0435-modifier-splattributes.md) and the
[{{on}} Modifier](0471-on-modifier.md) was introduced, there was no first-class
API for listening to DOM events on components you do not control. This required
the component authors to enumerate events they want to expose and accept them
as callback arguments, manually dispatching them in the `Ember.Component`
event-handling hooks.

The built-in components were historically bound by the same constraints here
as well. Before modifiers, the [documented way](https://github.com/emberjs/ember.js/blob/74a99a2971d327783b23a1093cf9564265db1a9d/packages/%40ember/-internals/views/lib/mixins/text_support.js#L87-L108)
of handling events on the built-in input components are via the "dasherized"
named arguments, such as `<Input @key-down={{this.myAction}} />`.

However, as mentioned above, due to the way arguments are handled, the event
handling hooks inherited from `Ember.Component` were frequently misused
instead, which would prevent these "dasherized" callbacks from running, as they
would overwrite the re-dispatching logic.

On top of this, there has also been [a bug](https://github.com/emberjs/ember.js/pull/18997)
that prevented these callback arguments from working reliably. Unfortunately,
this was not investigated in a timely manner and was only fixed very recently.
This nudged developers towards the incorrect approaches even more and further
propagated the confusion, some of which has made their way into the official
learning materials (this is currently being addressed).

* * *

In addition, for a variety of historical reasons, the implementation of the
built-in components, that is, the `Checkbox`, `LinkComponent`, `TextArea` and
`TextField` component classes were documented as public API, as well as the
`TextSupport` mixin, which is considered private but is well-documented and is
accessible through the `Ember` global.

Because of this, the fact that they are implemented as `Ember.Component`
subclasses are highly observable. Some common use cases and consequences are:

1. These classes can be imported for subclassing. For example, an app can
   export a custom subclass of `TextField` at, say, `app/components/my-text-field.js`
   which can then be invoked as <MyTextField>.

2. These classes can be reopened to modify the behavior of the built-in
   components. For example, this will add a CSS class to all `<Input>`
   components rendered in the app:

   ```js
   import TextField from '@ember/component/text-field';

   TextField.reopen({
     classNames: ['i-am-a-text-field']
   });
   ```

   Likewise, `reopenClass` can be used to modify the class itself.

3. Similar to the above, reopening the `Ember.Component` super class will
   affect all built-in components as well. For example, this will also add a
   CSS class to all built-in components rendered in the app:

   ```js
   import Component from '@ember/component';

   Component.reopen({
     classNames: ['i-am-a-component']
   });
   ```

   Likewise, `reopenClass` can be used to modify the super class itself, which
   will be inherited by the built-in component classes.

4. It is possible to lookup (or even replace) the built-in components using the
   `owner.factoryFor` and `owner.lookup` runtime APIs.

These are all opportunities for the internal implementation to leak, therefore
making changes to how the built-in components are implemented – specifically
stop subclassing from `Ember.Component` – will likely be a breaking change for
some, even if the new implementations otherwise supported all documented APIs
perfectly.

### The Plan

At their core, the designs of the built-in components are rather simple. Take
the `<Input>` component for example, its job is to provide a convince on top of
the native `<input>` element. It accepts `@type` and `@value` (or `@checked`)
as arguments and keeps the underlying `<input>` element in sync with the given
app state.

That description should cover 90% of what the average developer need to know
about this component. It should feel pretty similar to using any other modern
components in the Ember ecosystem – attributes can be passed in angle bracket
invocation, and native events can be listened to with the `{{on}}` modifier.

There is a little bit extra this component does – it takes callbacks for some
higher-level "logical events" such as `@enter` and `@escape-press`. Arguably
these aren't strictly necessary with the ubiquity of the `{{on}}` modifier or
specialized addons like [ember-keyboard](http://adopted-ember-addons.github.io/ember-keyboard/usage).
Perhaps we wouldn't have introduced these features today, but since there is
not a direct 1:1 translation to migrate away and there aren't really anything
wrong with them, we are not proposing to remove them to avoid needless churn.

Beyond that, everything else that is now redundant or obselete (such as named
arguments that exists purely for binding HTML attributes and callback arguments
that have corresponding native events) should be deprecated and removed.

Concretely, the plan is to:

1. Deprecate and remove any mechanisms that could leak the implementation
2. Deprecate and remove any features that are specific to `Ember.Component`
3. Deprecate and remove any redundant or obselete features

This RFC focuses on the first step, and the rest will be proposed separately
in follow-up RFC(s).

## Detailed design

This RFC proposes the following deprecations:

1. Deprecate importing the following modules
   1. `@ember/component/checkbox`
   2. `@ember/component/text-area`
   3. `@ember/component/text-field`
   4. `@ember/routing/link-component`
2. Deprecate accessing the following properties on the `Ember` global
   1. `Ember.Checkbox`
   2. `Ember.LinkComponent`
   3. `Ember.TextArea`
   4. `Ember.TextField`
   5. `Ember.TextSupport` (already private, no import path available)
3. Deprecate calling `reopen` or `reopenClass` on the classes and mixins listed
   above, whether they were obtained through an import or the `Ember` global
4. Deprecate calling `reopen` or `reopenClass` on the `Ember.Component` super
   class (which is also the default export of `@ember/component`), but not when
   called on a subclass of `Ember.Component` other than those listed above
5. Deprecate calling `lookup` or `factoryFor` on an `Owner` with the following
   specifiers
   1. `component:input`
   2. `component:link-to`
   3. `component:textarea`
   4. `component:-checkbox` (already considered private)
   5. `component:-text-field` (already considered private)
   6. `template:component/input`
   7. `template:component/link-to`
   8. `template:component/textarea`
   9. `template:component/-checkbox` (already considered private)
   10. `template:component/-text-field` (already considered private)
6. Deprecate overriding the following factories on an `Owner`, through placing
   files the corresponding locations in the `app` tree or through any other
   means such as runtime registrations or using a custom `Resolver`
   1. `component:-checkbox`
   2. `component:-text-field`
   3. `template:component/input`
   4. `template:component/link-to`
   5. `template:component/textarea`
   6. `template:component/-checkbox` (already considered private)
   7. `template:component/-text-field` (already considered private)

The first two sets of deprecations removes the classes themselves from being
public APIs.

In order to support apps that have implemented custom components by subclassing
these built-in classes, the current implementations will be moved to a legacy
addon and remain "frozen" in there. Future versions of Ember will stop basing
the built-in components on these legacy implementaitons, but custom subclasses
will continue to work. The deprecation message should provide information about
this legacy addon, or link to the deprecation details page that does.

The third and forth prevents globally modifiying the behavior of built-in
components. Users are encouraged to create wrapper components for use in their
apps or create custom subclasses using the legacy addon.

The fifth deprecation prevents leakage of the classes to runtime code, which is
complimentary to the first two sets of deprecations.

The last group of deprecation prevents partially replacing the built-in
components, such as replacing only the template but not the class.

Notably, the last group of deprecation does not prevent replacing the built-in
components in general, by defining a component with the same name. While this
is not necessarily _encouraged_ and would certainly be confusing for developers
and tools alike, we do not intend to "reserve" the built-in component names, so
if an app provides a component with the same name, they will take precedence
over the built-in components.

## How we teach this

The API documentation should be updated to document the built-in components as
components, not as classes. When reaidng the documentation, developers should
be able to understand the components in terms of what _arguments_ they are able
to pass. The exact format for documenting components is left unspecified to
provide the learning team some flexibility in accomplish this goal.

## Drawbacks

Apps that heavily customizes the built-in components will have to put in some
work to migrate, though this is largely mitigated by making the existing
implementations available through the legacy addon.

## Alternatives

* We can leave the existing built-in components around, ship new, modernized
  versions under new names, without deprecating the old ones. This will create
  more cruft and confusion in the long run, as may be difficult for developers
  to determine which ones are the recommended ones.

* We can leave the existing built-in components around, ship new, modernized
  versions under new names, and deprecating the old ones. Assuming the new
  components are roughly a subset of the existing ones, this will have more or
  less the same end result as the plan laid out above, but possibly with more
  intermediate churn.

  In this case, developers would have to figure out how to refactor away from
  legacy features that are no longer supported by the new version before they
  could fully migrate, similar to how developers are migrating from classic to
  Glimmer components today.

  By deprecating and removing individual features, as proposed in this plan, we
  will be able to provide more directly actionable guidance in each case.

* We can remove the built-in components altogether. This is a much bigger lift
  that would require analyising all existing use cases and likely requires
  designing new capabilities and features to fill some of those gaps.

  Other than the problems outlined in this RFC, the built-in components mostly
  get the job done adequately, and we feel like there is not a lot of value in
  taking on the project to reimagine them significantly and force the entire
  community to migrate to completely new patterns at this moment.

  Our preference would therefore be to pare down the existing components to
  address the problems raised here. In the meantime, we will work on providing
  primitives and capabilities that underpins these built-in components (such as
  [router helpers](0391-router-helpers.md)), so the community can experiment
  with alternatives and new patterns.

  If and when they mature to the point of becoming widely accepted as best
  practices in the ecosystem, we can revisit deprecating and removing any of
  these built-in components as they become redundant. Broadly speaking, we do
  not believe we are at that point yet, so in the meantime, we should not stop
  improving what we have.

## Unresolved questions

None.
