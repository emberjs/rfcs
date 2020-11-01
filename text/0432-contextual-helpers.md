---
Start Date: 2018-12-17
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/432
Tracking: https://github.com/emberjs/rfc-tracking/issues/6

---

# Contextual Helpers and Modifiers (a.k.a. "first-class helpers/modifiers")

## Summary

We propose to extend the semantics of Handlebars helpers and modifiers such
that they can be passed around as first-class values in templates.

For example:

```hbs
{{join-words "foo" "bar" "baz" separator=","}}

{{!-- ...is functionally equivalent to... --}}

{{#let (helper "join-words" separator=",") as |join|}}
  {{#let (helper join "foo") as |foo|}}
    {{#let (helper foo "bar") as |foo-bar|}}
      {{foo-bar "baz"}}
    {{/let}}
  {{/let}}
{{/let}}
```

```hbs
<Submit {{on
  click=(action "submit")
  mouseenter=(action "highlight")
  mouseleave=(action "unhighlight")
}} />

{{!-- ...is functionally equivalent to... --}}

{{#let (modifier "on") as |on|}}
  {{#let (modifier on click=(action "submit")) as |on-click|}}
    {{#let (modifier on-click mouseenter=(action "highlight")) as |on-click-enter|}}
      {{#let (modifier on-click-enter mouseleave=(action "unhighlight")) as |on-click-enter-leave|}}
        <Submit {{on-click-enter-leave}} />
      {{/let}}
    {{/let}}
  {{/let}}
{{/let}}
```

## Motivation

[RFC #64](https://github.com/emberjs/rfcs/pull/64) introduced a feature known
as "contextual components", which allowed components to be passed around as
first-class values. While this is a somewhat advanced feature, it allowed addon
authors to encapsulate internal state and logic, which in turns, allowed them
to create easy-to-use and easy-to-understand DSL-like APIs that could benefit
users of all level.

For example, the original RFC used form controls as a motivating example.
Without contextual components, an addon that provides form-building components
might have to expose an API like this:

```hbs
<SuperForm @model={{this.post}} as |f|>
  <SuperInput @model={{this.post}} @name="title" />
  <SuperTextarea @model={{this.post}} @name="body" />
  <SuperSubmit @form={{f}} />
</SuperForm>
```

As you can see, this is far from ideal for several reasons. First, to avoid
collision, the addon author had to prefix all the components. Second, the
`@model` argument has to be passed to all the controls that needs it. Finally,
in cases where the components need to communicate with each other (the form and
the submit button in the example), they would have to expose some internal state
(the `|f|` block param) that the user would have to manually thread through. Not
only does this make the API very verbose, it also breaks encapsulation.

Instead, the contextual components feature allows the addon author to expose an
API like this:

```hbs
<SuperForm @model={{this.post}} as |f|>
  <f.Input @name="title" />
  <f.Textarea @name="body" />
  <f.Submit />
</SuperForm>
```

Behind the scene, the `<SuperForm>` component's template would look something
like this:

```hbs
<form ...>
  {{yield (hash
    Input=(component "super-input" form=this model=this.model)
    Textarea=(component "super-textarea" form=this model=this.model)
    Submit=(component "super-submit" form=this model=this.model)
  )}}
</form>
```

Here, the `component` helper looked up the components by name (first argument)
and packaged up ("curried") any additional arguments (`form=this`) into an
internal object (known as a "component definition" in the Glimmer VM). This
object can then be passed around like any other values and invoked at a later
time.

(Typically, a number of them are passed to the `hash` helper to "bundle" them
into a single object, but this is not required.)

While this is indeed a pretty advanced feature, the users of `SuperForm` do not
need to be aware of these implementation details in order to use the addon.
This had proved to be a very useful and powerful feature and enabled a number
of popular addons, such as
[ember-cli-addon-docs](https://ember-learn.github.io/ember-cli-addon-docs/),
[ember-bootstrap](https://www.ember-bootstrap.com),
[ember-table](https://opensource.addepar.com/ember-table/),
[ember-paper](https://miguelcobain.github.io/ember-paper/),
[ember-power-calendar](https://ember-power-calendar.com),
[ember-accordion](http://khorus.github.io/ember-accordion/),
[emberx-select](https://emberx-select.netlify.com/),
[ember-light-table](https://offirgolan.github.io/ember-light-table/).

The original RFC left an important question unanswered – [should this feature
be available for helpers, too?](https://github.com/emberjs/rfcs/blob/master/text/0064-contextual-component-lookup.md#unresolved-questions)

In this RFC, we argue – yes, this feature will be just as useful for helpers as
well as modifiers.

For example, the `SuperForm` addon API can be expanded to include some extra
helpers and modifiers, like so:

```hbs
<SuperForm @model={{this.post}} as |f|>
  <f.Input @name="title" />

  {{!-- f.is-valid and f.error-for are contextual helpers --}}
  {{#unless (f.is-valid "title")}}
    <div class="error">This field {{f.error-for "title"}}</div>
  {{/unless}}

  {{!-- f.auto-resize is a contextual modifier --}}
  <f.Textarea @name="body" {{f.auto-resize maxHeight="500"}} />

  <f.Submit />
</SuperForm>
```

For reference, the `<SuperForm>` component's template would look something like
this:

```hbs
<form ...>
  {{yield (hash
    is-valid=(helper "super-is-valid" form=this model=this.model)
    error-for=(helper "super-error-for" form=this model=this.model)
    auto-resize=(modifier "super-auto-resize")
    ...
  )}}
</form>
```

This RFC proposes a complete design for enabling this capability.

## Detailed design

### The `helper` and `modifier` helpers

This RFC introduces two new helpers named `helper` and `modifier`, which work
similarly to the `component` helper:

* When passed a string (e.g. `(helper "foo")`) as the first argument, it will
  produce an opaque, internal "helper definition" or "modifier definition"
  object that can be passed around and invoked elsewhere.

* Any additional positional and/or named arguments (a.k.a. params and hash)
  will be stored ("curried") inside the definition object, such that, when
  invoked, these arguments will be passed along to the referenced helper or
  modifier.

Some additional details:

* When the first argument passed to the `helper` or `modifier` helper is
  `null`, `undefined` or an empty string, it will produce a no-op definition
  object. In the case of the `helper` helper, this will produce `undefined`
  when invoked, regardless of the arguments that are passed to the invocation.
  In the case of the `modifier` helper, it will not perform any operations on
  the target element.

* When the first argument passed to the `helper` or `modifier` helper is a
  string, it will be used to resolve a helper or modifier (respectively) with
  the same name. If the resolution failed, it will result in a runtime error.
  However, the timing of this lookup is unspecified. `(helper "not-a-helper")`
  may result in an immediate error, or it may happen when it is later passed
  into the `helper` helper a second time, or it may happen when it is invoked.
  If it is never invoked, the error may not be reported at all. This timing may
  change between releases and should not be relied upon.

* Some built-in helpers or modifiers may not be resolvable with the `helper`
  and `modifier` helpers. For example, `(helper "debugger")` and
  `(helper "yield")` will not work, as they are considered _keywords_. For
  implementation simplicity, we propose to forbid resolving built-in helpers,
  components and modifiers this way across the board (i.e. a runtime error).
  We acknowledge that there are good use cases for this feature such as
  currying the `array` and `hash` helpers, and will consider enabling them in
  the future on a case-by-case basis.

* Similarly, contextual helpers cannot be named after certain keywords. For
  example, `{{#let ... as |yield|}} {{yield}} {{/let}}` will not work. We
  propose to turn these cases into syntax errors.

* A contextual helper or modifier can be further "curried" by passing them back
  into the `helper` or `modifier` helper again, as shown in the example in the
  [summary](#Summary) section. This will produce a new definition object.

* When the first argument passed to the `helper` or `modifier` helper is a
  bound value, a new definition object will be produced whenever the value
  changes. This will _invalidate_ all downstream invocations. If the previous
  value is a [simple helper](https://emberjs.com/api/ember/3.7/functions/@ember%2Fcomponent%2Fhelper/helper),
  this has no observable effect and Ember will simply invoke the new helper
  value. If the previous value is a [class-based helper](https://emberjs.com/api/ember/3.7/classes/Helper),
  or a modifier, the existing instance will be destroyed before the new value
  is invoked. On the other hand, if only the curried arguments has changed, the
  helper or modifier instances (if any) will remain.

* An important implication of the teardown semantics is that it is possible for
  a modifier to be destroyed while its target element lives on for much longer.
  Therefore, it is important to actually teardown any event listeners and
  cleanup any associated states in the `destroyModifier` hook.

* Positional arguments are "curried" the same way as the `component` helper.
  This matches the behavior of `Function.prototype.bind`.

  ```hbs
  {{#let (helper "join-words" separator=",") as |join|}}
    {{join "foo" "bar" "baz"}} {{!-- "foo,bar,baz" --}}

    {{#let (helper join "foo") as |foo|}}
      {{foo "bar" "baz"}} {{!-- "foo,bar,baz" --}}

      {{#let (helper foo "bar") as |foo-bar|}}
        {{foo-bar "baz"}} {{!-- "foo,bar,baz" --}}
      {{/let}}

    {{/let}}

  {{/let}}
  ```

* Named arguments are curried the same way as the `component` helper.
  This matches the "last-write-wins" behavior of `Object.assign`.

  ```hbs
  {{#let (helper "join-words" "foo" "bar" "baz") as |join|}}
    {{join separator=","}} {{!-- foo,bar,baz --}}

    {{#let (helper join separator=",") as |comma|}}
      {{comma separator=" "}} {{!-- foo bar baz --}}

      {{#let (helper comma separator=" ") as |space|}}
        {{space separator="-"}} {{!-- foo-bar-baz --}}
      {{/let}}

    {{/let}}

  {{/let}}
  ```

* When a definition object is passed into JavaScipt (e.g. as an argument to a
  JavaScript helper), the resulting value is unspecified (hence "opaque"). In
  particular, for helpers, it is _not_ guarenteed that it will be an invokable
  JavaScript function. The only guarentee provided is that, when passed back
  into Handlebars it will be an invokable value. Hanging onto a definition
  object in JavaScript may result in unexpected memory leaks, as these objects
  may close over arbitrary template states.

### Invoking contextual helpers

Invoking a contextual helper is no different from invoking any other helpers:

```hbs
{{#let (helper "join-words" "foo" "bar" separator=" ") as |foo-bar|}}

  {{!-- content position --}}

  {{foo-bar}}

  {{foo-bar "baz"}}

  {{foo-bar separator=","}}

  {{!-- not necessary, but works --}}

  {{helper foo-bar}}

  {{helper foo-bar "baz"}}

  {{helper foo-bar separator=","}}

  {{!-- attribute position --}}

  <div class={{foo-bar}}>...</div>

  <div class={{foo-bar "baz"}}>...</div>

  <div class={{foo-bar separator=","}}>...</div>

  {{!-- not necessary, but works --}}

  <div class={{helper foo-bar}}>...</div>

  <div class={{helper foo-bar "baz"}}>...</div>

  <div class={{helper foo-bar separator=","}}>...</div>

  {{!-- curly invocation, argument position --}}

  {{my-component value=(foo-bar)}}

  {{my-component value=(foo-bar "baz")}}

  {{my-component value=(foo-bar separator=",")}}

  {{!-- these will pass the helper itself into the component, instead of invoking it now --}}

  {{my-component helper=foo-bar}}

  {{my-component helper=(helper foo-bar)}}

  {{my-component helper=(helper foo-bar "baz")}}

  {{my-component helper=(helper foo-bar separator=",")}}

  {{!-- angle bracket invokation, argument position --}}

  <MyComponent @value={{(foo-bar)}} />

  <MyComponent @value={{foo-bar "baz"}} />

  <MyComponent @value={{foo-bar separator=","}} />

  {{!-- these will pass the helper itself into the component, instead of invoking it now --}}

  <MyComponent @helper={{foo-bar}} />

  <MyComponent @helper={{helper foo-bar}} />

  <MyComponent @helper={{helper foo-bar "baz"}} />

  <MyComponent @value={{helper foo-bar separator=","}} />

  {{!-- sub-expression positions --}}

  {{yield (foo-bar)}}

  {{yield (foo-bar "baz")}}

  {{yield (foo-bar separator=",")}}

  {{!-- these will yield the helper itself ("contextual helper"), instead of invoking it now --}}

  {{yield foo-bar}}

  {{yield (helper foo-bar)}}

  {{yield (helper foo-bar "baz")}}

  {{yield (helper foo-bar separator=",")}}

  {{!-- deeply nested sub-expression --}}

  {{#if (eq (concat ">>> " (foo-bar "baz") " <<<") ">>> foo bar baz <<<")}}
    This is true.
  {{/if}}

  {{!-- runtime error: not a component --}}
  <foo-bar />

  {{!-- runtime error: not a modifier --}}
  <div {{foo-bar}}>
{{/let}}
```

### Invoking contextual modifiers

Invoking a contextual helper is no different from invoking any other modifiers:

```hbs
{{#let (modifier "on" click=(action "submit")) as |on-click|}}

  {{!-- HTML elements --}}

  <button {{on-click}}>Click Me!!</button>

  <button {{on-click "extra" "args"}}>Click Me!!</button>

  <button {{on-click mouseenter=(action "highlight") mouseleave=(action "unhighlight")}}>
    Click Me!!
  </button>

  {{!-- not necessary, but works --}}

  <button {{modifier on-click}}>Click Me!!</button>

  <button {{modifier on-click "extra" "args"}}>Click Me!!</button>

  <button {{modifier on-click mouseenter=(action "highlight") mouseleave=(action "unhighlight")}}>
    Click Me!!
  </button>

  {{!-- components --}}

  <MyComponent {{on-click}} />

  <MyComponent {{on-click "extra" "args"}} />

  <MyComponent {{on-click mouseenter=(action "highlight") mouseleave=(action "unhighlight")}} />

  {{!-- not necessary, but works --}}

  <MyComponent {{modifier on-click}} />

  <MyComponent {{modifier on-click "extra" "args"}} />

  <MyComponent {{modifier on-click mouseenter=(action "highlight") mouseleave=(action "unhighlight")}} />

  {{!-- these will pass the modifier itself into the component, instead of invoking it now --}}

  <MyComponent @modifier={{on-click}} />

  <MyComponent @modifier={{modifier on-click}} />

  <MyComponent @modifier={{modifier on-click "extra" "args"}} />

  <MyComponent @modifier={{modifier on-click mouseenter=(action "highlight") mouseleave=(action "unhighlight")}} />

  {{my-component modifier=on-click}}

  {{my-component modifier=(modifier on-click)}}

  {{my-component modifier=(modifier on-click "extra" "args")}}

  {{my-component modifier=(modifier on-click mouseenter=(action "highlight") mouseleave=(action "unhighlight"))}}

  {{!-- these will yield the modifier itself ("contextual modifier"), instead of invoking it now --}}

  {{yield on-click}}

  {{yield (modifier on-click)}}

  {{yield (modifier on-click "extra" "args")}}

  {{yield (modifier on-click mouseenter=(action "highlight") mouseleave=(action "unhighlight"))}}

  {{!-- runtime error: cannot invoke a modifier as a helper --}}

  {{yield (on-click)}}

  {{yield (on-click "extra" "args")}}

  {{yield (on-click mouseenter=(action "highlight") mouseleave=(action "unhighlight"))}}

  {{!-- runtime error: cannot append a modifier --}}

  {{on-click}}

  {{on-click "extra" "args"}}

  {{on-click mouseenter=(action "highlight") mouseleave=(action "unhighlight")}}

  {{!-- runtime error: cannot set an attribute to a modifier --}}

  <div class={{on-click}} />

  <div class={{on-click "extra" "args"}} />

  <div class={{on-click mouseenter=(action "highlight") mouseleave=(action "unhighlight")}} />

  {{!-- runtime error: not a component --}}
  <on-click />
{{/let}}
```

### Relationship with globals

Today, Ember apps rely heavily on the global namespace as the main mechanism of
making components, helpers and modifiers available. Ideally, in a world where
"everything is a value", the global and local namespace should behave the same
way. Global components, helpers and modifiers should just be global variables
that are implicitly defined around every templates in the app.

In other words, it is as if every template has the following hidden "prelude"
around its content:

```hbs
{{!-- prelude --}}
{{#let (component "input") as |input|}}
  {{#let (helper "concat") as |concat|}}
    {{#let (modifier "action") as |action|}}
      {{!-- ...other global components, helpers and modifiers omitted... --}}

      {{!-- begin template content --}}
      Your name:
      {{concat this.firstName " " this.lastName}}

      Change it:
      {{input value=this.firstName}}
      {{input value=this.lastName}}

      <button {{action "submit"}}></button>
      {{!-- end tempalte content ---}}
    {{/let}}
  {{/let}}
{{/let}}
```

While this largely matches how things work today, there are a few notable
differences where globals behave "unexpectedly" in this world.

First of all, it is not possible to reference a component, helper or modifier
in templates without invoking them today:

```hbs
{{!-- if `join-words` is a global helper, this works as expected --}}
{{!-- this invokes the helper and yield the result --}}

{{yield (join-words "foo" "bar" separator=",")}}
         ~~~~~~~~~~

{{!-- however, in this position, Ember does not "see" the helper --}}
{{!-- this falls back to looking up the `join-words` property on `this` --}}

{{yield join-words}}
        ~~~~~~~~~~

{{!-- as opposed to a "true" variable/value binding... --}}
{{!-- this yields the helper as a value, as expected --}}

{{#let (helper "join-words") as |join-words|}}
  {{yield join-words}}
          ~~~~~~~~~~
{{/let}}
```

In the long term, we propose to unify the semantics such that globals will
behave exactly like local bindings (i.e. we should make this second case work).

However, is not possible in the short term. This is due to the ambiguity
between referencing a global variable and the
[property lookup fallback](https://github.com/emberjs/rfcs/blob/master/text/0308-deprecate-property-lookup-fallback.md)
feature. We propose we simply wait until the property lookup fallback is fully
deprecated and removed, at which point we can reclaim the syntax.

In the mean time, globals can be referenced explicitly using the `component`,
`helper`, and `modifier` helpers.

Another difference is how global helpers can be invoked without arguments in
named arguments position for angle bracket invocations:

```hbs
{{!-- if `pi` is a global helper that returns the value of the constant 𝛑 --}}
{{!-- this invokes the helper and passes the value 3.1415... --}}
<MyComponent @value={{pi}} />
                      ~~

{{!-- as opposed to a "true" variable/value binding... --}}

{{#let (helper "pi") as |pi|}}
  {{!-- this passes the helper into the component --}}
  <MyComponent @value={{pi}} />
                        ~~

  {{!-- this invokes the helper and passes the value 3.1415... --}}
  <MyComponent @value={{(pi)}} />
                        ~~~~
{{/let}}
```

Notably, this problem only exists in the named arguments position and only in
angle bracket invocations. For content and attribute positions, it would not
make sense to pass a helper "by value", so the ambiguity does not exist (so it
always invokes). For sub-expression positions (which includes argument
positions for curly invocations), the parentheses are already mandatory
(otherwise it invokes the property fallback).

We propose to deprecate auto-invoking global helpers with no arguments in named
argument positions for angle bracket invocations and require the parentheses
instead. This will make room for unifying the global semantics in the future.

It is also worth pointing out that, since helpers tend to be pure, helpers
that take no arguments are exceedingly rare.

Finally, the last challenge to the unification is it is entirely possible to
have any combinations of components, helpers and modifiers all with the same
name today. This works, as they currently live in different "namespaces", and
each lookup is contextually scoped to their respective "namespace" depending on
the position where it is invoked:

```hbs
{{!-- foo-bar, the modifier here --}}
<div {{foo-bar}} />
       ~~~~~~~

{{!-- foo-bar, the helper here --}}
<div class={{foo-bar}} />
             ~~~~~~~

{{!-- foo-bar, the helper here --}}
{{#let (foo-bar) as |result|}}
        ~~~~~~~
{{/let}}

{{!-- foo-bar, the component here --}}
{{#foo-bar}}...{{/foo-bar}}
   ~~~~~~~        ~~~~~~~

{{!-- prefers foo-bar, the component here --}}
{{!-- if not found, then foo-bar, the helper --}}
{{foo-bar}}
  ~~~~~~~
```

Since this could get pretty confusing, most developers already avoid giving
these unrelated the same names. However, it is certainly possible that they
may happen by accident and go unnoticed (e.g. an addon introducing a global
helper that "conflicts" with a component in the app).

These kind of naming conflicts would not make sense in the value-based world.
Imagine if this is how JavaScript works:

```js
class FooBar {}

function FooBar() {}

const FooBar = 1;

class FooBarBaz extends FooBar {}
//                      ~~~~~~ FooBar, the class here

console.log(FooBar());
//          ~~~~~~ FooBar, the function here

console.log(FooBar + 1);
//          ~~~~~~ FooBar, the constant here

someObj.FooBar = FooBar;
//               ~~~~~~ well, which one is this?
```

Clearly, this would be unacceptable and is similar to the situation we find
ourselves in here.

We propose to issue deprecation warnings whenever we detect these conflicts,
both at build time and at invocation time, while maintaining the same lookup
precedence for the time being. For example, when invoking a component in the
content position, if we see that there is also a helper with the same name, it
should result in a deprecation asking the developer to remain one or the other.

Notably, there is such a conflict in Ember today where `action` is both a
helper and a modifier. Instead of deprecating one of them, we propose to use an
internal mechanism to produce a single special value such that it will be
invokable as either a modifier or a helper context. This is different than
"namespace" semantics in that there is only one context-independent value in this
special case, i.e. `(helper "action") === (modifier "action")`.

We also acknowledge that, so long as there are _implicit_ globals, we may never
be able to truly unify global bindings with local ones, as implicit global
bindings have a high risk of conflicting with HTML elements. Consider the
built-in `input` helper, or an in-app `main` helper. If these were implicitly
turned into global identifiers, they would conflict with the HTML elements with
the same name:

```hbs
<input type="text">
 ~~~~~ now refers to the global `input` identifier?

<main>...</main>
 ~~~~ now refers to the global `main` identifier?
```

While the problem exists for local bindings also, it was already addressed in
[RFC #311](https://github.com/emberjs/rfcs/pull/311). With local bindings, this
problem is fairly noticible and understandable since the conflict is introduced
nearby. The solution is also fairly simple – just rename the local variable to
avoid the conflict. With proper linting, this could be quite easily avoided
altogether.

With _implicit_ global bindings, this problem is much more difficult to spot
and reason about. There is also no quick way out, other than to rename the
global component, helper or modifer which could be difficult or not an option
at all for addon authors trying to maintain compatibility. To truly resolve
this conflict, we would have to eliminate implicit globals, which is out of
scope for this RFC. This also wouldn't be a problem until all the proposed
deprecations are implemented and removed, which would be quite some time.

We _speculate_ that when all the of that is said and done, we would have an
alternative resolution mechanism ("template imports") that does not have this
problem. Alternatively, we could exclude the angle bracket invocation position
from being able to "see" implicit global identifiers.

### Local helpers and modifiers

A nice fallout of this plan is that developers will be able to define helpers
and modifiers specific to a component locally in the same JavaScript file:

```js
// app/components/date-picker.js

import Component from '@ember/component';
import { helper } from '@ember/component/helper';

export default Component.extend({
  date: null, // passed in

  'format-date': helper(function(params, hash) {
    /* ... */
  })
});
```

```hbs
{{!-- app/templates/components/date-picker.hbs --}}
{{this.format-date this.date}}
```

In additional to encapsulation and namespacing, this will also enable even more
advanced use cases that uses the component's state:

```js
// app/components/filtered-each.js

import Component from '@ember/component';
import { helper } from '@ember/component/helper';
import { computed } from '@ember/object';

export default Component.extend({
  list: null, // passed in
  callback: null, // passed in

  filter: computed('callback', function() {
    return helper(params => this.callback(params[0]));
  });
});
```

```hbs
{{!-- app/templates/components/filtered-each.hbs --}}

{{#each this.list as |item|}}
  {{#if (this.filter item)}}
    {{yield item}}
  {{/if}}
{{/each}}
```

This feature would work with element modifiers as well.

Ideally, this should also work with components. However, currently there are
two pieces to a component – a template and a JavaScript class, either could be
optional. This poses a challenge to invoking components this way – without
going through the component helper, there is no easy way to import or package
a component into a single value. This is a solvable problem, but to design a
solution for that would be out of scope for this RFC. For the time being, the
only way to get a handle of a "component defition value" would be through the
component helper. Attempting to "invoke" just the component template or class
this way will result in a runtime error.

## How we teach this

There are two sides to this feature – the consumption side and the authoring
side.

The consumption side refers to learning how the contextual helper and modifier
values can be used (invoked). We expect developers to enounter this mainly
through addons that others have written. So long as there is adequate
documentation from the addon authors, we expect that this group of users can be
immediately productive by simply treating these APIs as DSLs, similar to the
Router DSL.

In other words, while this group of developer may not immediately understand
how to _author_ these kind of APIs, or what is involved under-the-hood to make
it work, the design goal is that it should feel straightforward to _consume_
this style of API.

The authoring side refers to using the `helper` and `modifier` helpers, and
more importantly, the advanced composition patterns that motivated their
existence in the first place As with the `component` helper and other
"higher-order functions" in JavaScript, this is a somewhat advanced topic that
is mainly targeted at addon authors and advanced developers.

For this group of users, we expect this feature to complement and complete the
"contextual components" feature. Developers who are already familiar with that
feature should feel right at home. We expect to be able to introduce this new
feature at the point where we currently teach contextual components today.

In the long term, the unifications proposed in this RFC should make these
concepts easier to teach for this group of developers, as components, helpers
and modifiers, whether global or contextual, will all behave uniformly. The
value-based semantics also better matches JavaScript which they are probably
already familiar with.

The official documentations should be updated to include this feature:

* The new `helper` and `modifier` helpers need be be added to the API docs.
  We should consider cross-linking between the `helper`, `modifier` and
  `component` helpers since they solve a similar problem.

* The guide should be updated to teach this feature as well. We recommend
  teaching the two sides of the feature separately, and prioritize the
  consumption side, as that is what beginners are likely to encounter first.

  For example, when teaching component invocations, there can be a section that
  mentions:

  > Sometimes, components may be yielded to you as a block param. These are
  called contextual components, and they can be invoked just like any other
  components you have encountered so far.
  >
  > ...examples...
  >
  > To learn how to do this yourself, skip ahead to the "Composition Patterns"
  > section (link).

* For the authoring side, we recommend teaching the helper, modifier and
  component version of the feature in a single place (such as a "Composition
  Patterns" section), cross-linked from their respective sections, rather than
  repeating it three times.

## Drawbacks

This RFC introduces another feature that developers may encounter and have to
learn when consuming addons. However, on the whole, we think this will simplify
things more than adding to the concepts – as it ultimiately try to unify the
behavior of components, helpers and modifiers (and in the future, globals).
This should make things feel more consistent and allow developers to apply
their knowledge consistently across the board.

In the short term, this feature may amplify some of the mismatches and causes
confusions where the legacy semantics does not perfectly match the new world we
are building. This could be mitigated with helpful deprecation messages.

## Alternatives

[RFC #208](https://github.com/emberjs/rfcs/pull/208) has previously explored
the same design space. It solves the same fundamental problems, but proposes
two seperate helpers resolution/currying and invocation. This is largely due to
limitations and ambiguities in Handlebars. This RFC attempts to remove the need
of a separate invocation helper by resolving the ambiguities and integrating
more tightly with Handlebars. If accepted, this RFC will supersede the design
proposed in RFC #208.

As proposed, this RFC relies heavily on context-dependent syntatic positions to
disambiguate between component, helper and modifier invocations. For example,
while they may look similar, the following syntax does not produce the same
result:

```hbs
{{foo-bar baz="bat"}}

{{(foo-bar baz="bar")}}
```

If `foo-bar` is a helper, either would work. However, if `foo-bar` is a
component, only the first form would work and the second form would result in
a runtime error (trying to invoke a component as helper).

A different design has been considered where the first form is just strictly a
syntactic sugar for the latter and `(...)` invocation is one true primitive
that ties everything together.

Specifically, when "invoked" with `(...)`, a component or modifier simply
produces a value, which is a definition object with the curried arguments, i.e.
`(...)` is a syntatic surgar for currying using the `helper` and `modifier`
helpers. The `{{...}}` syntax then simply "append" the curried definition
object by first invoking it.

This design turned out to add more complexities and confusions than the
unification has brought to the table, and so that design was abandoned in favor
of what is proposed here.

Another alternative is to keep the global namespace separate from the local
namespace, thus avoiding the need for most deprecations. In practice, we
believe this would result in much more confusion when things do not behave the
way you would expect, but only in some niche corner cases.

## Unresolved questions

None
