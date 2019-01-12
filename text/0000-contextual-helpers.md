- Start Date: 2018-12-17
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Contextual Helpers (a.k.a. "first-class helpers")

## Summary

We propose to extend the semantics of Handlebars helpers such that they can be
passed around as first-class values in templates.

For example:

```hbs
{{join "foo" "bar" "baz" seperator=" "}}

{{!-- ...is functionally equivilant to... --}}

{{#let (helper "join" seperator=" ") as |join|}}
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

{{!-- ...is functionally equivilant to... --}}

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
in cases where the compoments need to communicate with each other (the form and
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

Behind the scene, the `<SuperForm>` compoment's template would look something
like this:

```hbs
<form ...>
  {{yield (hash
    Input=(component "super-input" form={{this}} model={{this.model}})
    Textarea=(component "super-textarea" form={{this}} model={{this.model}})
    Submit=(component "super-submit" form={{this}})
  )}}
</form>
```

Here, the `component` helper looked up the components by name (first argument)
and packaged up ("curried") any additional arguments (`form={{this}}`) into an
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

  <F.Submit />
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

* Any additional positional and/or named arguments will be stored ("curried")
  inside the definition object, such that, when invoked, these arguments will
  be passed along to the referenced helper or modifier.

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
  `(helper "yield")` will not work, as they are considered _keywords_.

* Similarly, contextual helpers cannot be named after keywords. For example,
  `{{#let ... as |yield|}} {{yield}} {{/let}}` will not work. We propose to
  turn these cases into syntax errors.

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
  {{#let (helper "concat") as |my-concat|}}
    {{my-concat "foo" "bar" "baz"}} {{!-- "foobarbaz" --}}

    {{#let (helper concat "foo") as |foo|}}
      {{foo "bar" "baz"}} {{!-- "foobarbaz" --}}

      {{#let (helper foo "bar") as |foo-bar|}}
        {{foo-bar "baz"}} {{!-- "foobarbaz" --}}
      {{/let}}

    {{/let}}

  {{/let}}
  ```

* Named arguments are curried the same way as the `component` helper.
  This matches the "last-write-wins" behavior of `Object.assign`.

  ```hbs
  {{#let (helper "hash") as |my-hash|}}
    {{my-hash value="foo"}} {{!-- hash with value="foo" --}}

    {{#let (helper my-hash value="foo") as |foo|}}
      {{foo value="bar"}} {{!-- hash with value="bar" --}}

      {{#let (helper foo value="bar") as |bar|}}
        {{bar value="baz"}} {{!-- hash with value="baz" --}}
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
{{#let (helper "join" "foo" "bar" seperator=" ") as |foo-bar|}}

  {{!-- content position --}}

  {{foo-bar}}

  {{foo-bar "baz"}}

  {{foo-bar seperator=","}}

  {{!-- not necessary, but works --}}

  {{helper foo-bar}}

  {{helper foo-bar "baz"}}

  {{helper foo-bar seperator=","}}

  {{!-- attribute position --}}

  <div class={{foo-bar}}>...</div>

  <div class={{foo-bar "baz"}}>...</div>

  <div class={{foo-bar seperator=","}}>...</div>

  {{!-- not necessary, but works --}}

  <div class={{helper foo-bar}}>...</div>

  <div class={{helper foo-bar "baz"}}>...</div>

  <div class={{helper foo-bar seperator=","}}>...</div>

  {{!-- curly invocation, argument position --}}

  {{my-component value=(foo-bar)}}

  {{my-component value=(foo-bar "baz")}}

  {{my-component value=(foo-bar seperator=",")}}

  {{!-- these will pass the helper itself into the component, instead of invoking it now --}}

  {{my-component helper=foo-bar}}

  {{my-component helper=(helper foo-bar)}}

  {{my-component helper=(helper foo-bar "baz")}}

  {{my-component helper=(helper foo-bar seperator=",")}}

  {{!-- angle bracket invokation, argument position --}}

  <MyComponent @value={{(foo-bar)}} />

  <MyComponent @value={{foo-bar "baz"}} />

  <MyComponent @value={{foo-bar seperator=","}} />

  {{!-- these will pass the helper itself into the component, instead of invoking it now --}}

  <MyComponent @helper={{foo-bar}} />

  <MyComponent @helper={{helper foo-bar}} />

  <MyComponent @helper={{helper foo-bar "baz"}} />

  <MyComponent @value={{helper foo-bar seperator=","}} />

  {{!-- sub-expression positions --}}

  {{yield (foo-bar)}}

  {{yield (foo-bar "baz")}}

  {{yield (foo-bar seperator=",")}}

  {{!-- these will yield the helper itself ("contextual helper"), instead of invoking it now --}}

  {{yield foo-bar}}

  {{yield (helper foo-bar)}}

  {{yield (helper foo-bar "baz")}}

  {{yield (helper foo-bar seperator=",")}}

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

### Ambigious invocations (invoking without arguments)

When invoking a contextual helper without arguments, the invocation becomes
ambigious. Consider this:

```hbs
{{#let (helper "concat" "foo" "bar") as |foo-bar|}}
  {{!-- content position --}}

  {{!-- does this render "foobar", or something like "[object Object]" ? --}}
  {{foo-bar}}

  {{!-- attribute position --}}

  {{!-- class="foobar", or something like class="[object Object]" ? --}}
  <div class={{foo-bar}}>...</div>

  {{!-- curly invokcation, argument position --}}

  {{!-- is this.value the string "foobar", or the "helper definition" ? --}}
  {{MyComponent value=foo-bar}}

  {{!-- angle bracket invokation, argument position --}}

  {{!-- is @value the string "foobar", or the "helper definition" ? --}}
  <MyComponent @value={{foo-bar}} />

  {{!-- sub-expression positions --}}

  {{!-- are we yielding the helper, or the string "foobar" ? --}}
  {{yield (hash foo-bar=foo-bar)}}

  {{!-- is this true or false? --}}
  {{#if (eq foo-bar "foobar")}}...{{/if}}
{{/let}}
```

In these cases, these are ambiguities between passing the _result_ of the helper
(first invoking the helper, then pass along the result), or passing the helper
itself (the "helper definition" object, so that it can be invoked or "curried"
again on the receiving side).

Since this RFC proposes to treat helpers and modifiers as first-class values,
they should generally be passed through as _values_. This is particularly
important in arguments and sub-expression positions. To invoke the helper, the
explicit `(some-helper)` syntax can be used instead:

```hbs
{{#let (helper "concat" "foo" "bar") as |foo-bar|}}
  {{!-- curly invokcation, argument position --}}

  {{!-- the component will receive the "helper definition" --}}
  {{MyComponent value=foo-bar}}

  {{!-- the component will receive the string "foobar" --}}
  {{MyComponent value=(foo-bar)}}

  {{!-- angle bracket invokation, argument position --}}

  {{!-- @value will be the "helper definition" --}}
  <MyComponent @value={{foo-bar}} />

  {{!-- @value will be the string "foobar" --}}
  <MyComponent @value={{(foo-bar)}} />

  {{!-- sub-expression positions --}}

  {{!-- yielding the helper --}}
  {{yield (hash foo-bar=foo-bar)}}

  {{!-- yielding the string "foobar" --}}
  {{yield (hash foo-bar=(foo-bar))}}

  {{!-- false --}}
  {{#if (eq foo-bar "foobar")}}...{{/if}}

  {{!-- true --}}
  {{#if (eq (foo-bar) "foobar")}}...{{/if}}
{{/let}}
```

However, in the case of content and attribute positions, it would be overly
pendantic to insist on the `{{(some-helper)}}` syntax, as the alternative of
printing `[object Object]` to the DOM is almost certainly not what the
developer had in mind. Therefore, we propose to allow contextual helpers to be
auto-invoked in these positions.

```hbs
{{#let (helper "concat" "foo" "bar") as |foo-bar|}}
  {{!-- content position --}}

  {{!-- "foobar" --}}
  {{foo-bar}}

  {{!-- not necessary, but works --}}
  {{(foo-bar)}}

  {{!-- attribute position --}}

  {{!-- class="foobar" --}}
  <div class={{foo-bar}}>...</div>

  {{!-- class="foobar" --}}
  <div class={{(foo-bar)}}>...</div>
{{/let}}
```

It should also be noted that modifiers doe not have the same problem, since
there are no other possible meaning in that position:

```hbs
{{#let (modifier "foo-bar") as |foo-bar|}}
  {{!-- modifier position: not ambigious --}}
  <div {{foo-bar}} />

  {{!-- not necessary, but works --}}
  <div {{(foo-bar)}} />

  {{!-- undefined behavior: runtime error or [object Object] ? --}}
  {{foo-bar}}

  {{!-- undefined behavior: runtime error or [object Object] ? --}}
  <div class={{foo-bar}} />
{{/let}}
```

### Deprecation

In today's Ember, "global helpers" (as opposed to "contextual helpers") do
not always follow the rules laid out above. In particular, they do not behave
the same way in arguments and subexpression positions:

```hbs
{{#let (helper "concat") as |my-concat|}}
  {{!-- curly invocation, argument position --}}

  {{!-- this.value is the helper --}}
  {{MyComponent value=my-concat}}

  {{!-- this.value is the undefined --}}
  {{MyComponent value=concat}}

  {{!-- angle bracket invocation, argument position --}}

  {{!-- @value is the helper --}}
  <MyComponent @value={{my-concat}} />

  {{!-- @value is an empty string (invoking concat with no arguments) --}}
  <MyComponent @value={{concat}} />

  {{!-- sub-expression positions --}}

  {{!-- yielding the helper --}}
  {{yield (hash value=my-concat)}}

  {{!-- yielding undefined --}}
  {{yield (hash value=concat)}}

  {{!-- false: it compares the helper with undefined --}}
  {{#if (eq my-concat concat)}}...{{/if}}
{{/let}}
```

This illustrates the problem: "global helpers" are not modelled as first-class
values today, they exists in a different "namespace" distinct from the "local
variables" namespace.

While this is not so different from other programming languages like Java and
Ruby which also do not treat functions (methods) as first-class values, it is
distinctly different from JavaScript which does. For example, `alert("hello")`
is the same as `let a = alert; a("hello");`, whereas in Java and Ruby (and
today's Handlebars), the moral equivalent of the latter would fail with `alert`
being an undefined reference.

The goal of this RFC is to iterate the Ember Handlebars programming model
towards a world closer to JavaScript's, where global names exists in the same
namespace as local names. We propose to deprecate all cases where global names
("global helpers", "global modifiers" and "global components") can be observed
to behave differently.

Specifically:

```hbs
{{#let (helper "concat") as |my-concat|}}
  {{!-- curly invocation, argument position --}}

  {{!-- deprecation: use `this.concat` instead --}}
  {{MyComponent value=concat}}

  {{!-- angle bracket invocation, argument position --}}

  {{!-- deprecation: use `{{(concat)}}` instead --}}
  <MyComponent @value={{concat}} />

  {{!-- sub-expression positions --}}

  {{!-- deprecation: use `this.concat` instead --}}
  {{yield (hash value=concat)}}

  {{!-- deprecation: use `this.concat` instead --}}
  {{#if (eq my-concat concat)}}...{{/if}}
{{/let}}
```

Overall, we expect the effect of this deprecation to be quite minimal. For the
cases that trigger a property lookup today, they are already covered in the
[Property Lookup Fallback Deprecation RFC](https://github.com/emberjs/rfcs/blob/master/text/0308-deprecate-property-lookup-fallback.md),
plus it would already be quite confusing to name an instance variable after a
global helper. For the cases where the "global helper" is implicitly invoked
without arguments, since helpers are supposed to be pure computations, a helper
that doesn't accept any arguments has very limited utility thus should also be
quite rare.

In addition, another natural fallout of this plan is that it is not be possible
to have helpers, components or modifiers with the same name, as they ultimately
will share the same namesapce. These conflicts will need to be detected and
deprecated as well.

### Local helpers (and modifiers)

A nice fallout of this plan is that developers will be able to define helpers
specific to a component locally (i.e. in the same JavaScript file):

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

{{input value=(this.format-date this.date)}}
```

In additional to encapsulation and namespacing, this will also enable even more
advanced stateful use cases:

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

## How we teach this

There are two sides to this feature.

First, there is the `helper` and `modifier` helpers that produces the "curried
values". As with the `component` helper and other "higher-order functions" in
JavaScript, this is a somewhat advanced concept that is mainly targeted at
addon authors and advanced developers.

For this group of users, we expect this feature to complement and complete the
"contextual components" feature. Developers who are already familiar with that
feature should feel right at home. We expect to be able to introduce this new
feature at places where we currently teach contextual components.

The second side of this feature is the invoking side. We expect intermediate
developers (and perhaps even beginners) to enounter this mainly through addons
that other developers have written. So long as there is adequate documentation
from the addon authors, we expect that this group of users can be immediately
productive by simply treating these APIs as DSLs (similar to the Router DSL).

In other words, while this group of developer may not immediately understand
how to _author_ these kind of APIs, or what is involved under-the-hood to make
it work, the design goal is that it should feel straightforward to _consume_
this style of API.

### Rationalization

We propose to rationalize the existing and proposed semantics into a coherent
model at a deeper level. This knowledge is necessary for day-to-day use, but
could be helpful for guiding the implementation as well as design of future
feature proposals.

On the high-level, the guiding principles are:

1. Components, helpers and modifiers are first-and-foremost values that are
   bound to Handlebars identifiers. When referring to these identifiers, they
   should consistently behave as inert values unless they are _explicitly_
   invoked.

2. Without violating the first principle, for ergonomic reasons, in places
   where it unambigiously would not make sense for them to behave as values,
   they should be _implicitly_ invoked rather than raising an error or giving
   nonsensical results.

For helpers, the explicit invocation syntax is `(...)`, i.e. `(foo-bar)`,
`(foo-bar "baz")`, `(foo-bar baz="bat")`, etc. This is already mandatory in all
sub-expression posisitions today.

It follows that the explicit syntax for invoking a helper and appending its
result to the DOM would be:

```hbs
<ul>
  <li>No arguments: {{(foo-bar)}}</li>
  <li>Positional argument: {{(foo-bar "baz")}}</li>
  <li>Named argument: {{(foo-bar baz="bat")}}</li>
</ul>
```

The `{{(...)}}` form results in a syntax error today. For completeness, we
propose to modify the grammar to allow this explicit form. However, it is
neither required nor necessarily encouraged, as it adds a lot of visual noise
to the template. Following the second guiding principle, the implicit helper
invocation syntax will continue to work:

```hbs
<ul>
  <li>No arguments: {{foo-bar}}</li>
  <li>Positional argument: {{foo-bar "baz"}}</li>
  <li>Named argument: {{foo-bar baz="bat"}}</li>
</ul>
```

The last two forms (with arguments) are non-ambigious: they could not possibly
mean anything else other than to invoke `foo-bar` with the given arguments.
Therefore, the parentheses will be automatically inserted at
parsing time in these cases.

The first form (without arguments) is indeed ambigious – in fact it will be a
violation of the first guiding principle if this is interpreted as anything
other than referring to the value. To be precise, it has to mean: "append the
value referred to by the `foo-bar` identifier (which happens to be a helper) as
content into the DOM". Therefore, we propose to rationalize this case
differently.

Indeed, the helper is passed as a value here, as opposed to being invoked.
However, at runtime, the rendering engine has to decide exactly how to append a
particular value, which happens to be a helper here, as content into the DOM.

There are several options here:
1. Raising an error ("I don't know how to turn a helper into content").
2. Using JavaScript's default `toString` (resulting in "[object Object]").
3. Invoking the helper with no arguments and then appending the _result_ as content.

We argue that the first option is too pedantic, the second option is not useful and
therefore the third option is unambigiously the only reasonable option.
This allows all three existing form to work while still staisfying the guiding principles.

This works similarly in attribute positions too.

Explicit invocation forms:

```hbs
<ul>
  <li class={{(foo-bar)}}>No arguments</li>
  <li class={{(foo-bar "baz")}}>Positional argument</li>
  <li class={{(foo-bar baz="bat")}}>Named argument</li>
</ul>
```

Implicit invocation forms:

```hbs
<ul>
  <li class={{foo-bar}}>No arguments</li>
  <li class={{foo-bar "baz"}}>Positional argument</li>
  <li class={{foo-bar baz="bat"}}>Named argument</li>
</ul>
```

Here, the first form relies on the runtime to invoke the helper before it is
added to the DOM, and the last two rely on automatic parentheses insertion
at parse time.

For named arguments position, it is slightly different.

Explicit invocation forms:

```hbs
<MyItem @name={{(foo-bar)}}>No arguments</MyItem>
<MyItem @name={{(foo-bar "baz")}}>Positional argument</MyItem>
<MyItem @name={{(foo-bar baz="bat")}}>Named argument</MyItem>
```

While the parentheses in the last two forms can be unambigiously omitted, the
same is not true about the first form. It is conceivable that there is both the
need to pass helpers as value and the results of their invocations. Therefore,
when there are no arguments, the parentheses are mandatory for invocations to
disambiguate between the two.

For components, the explict invocation syntax is `<...>`, i.e. `<FooBar />`,
`<FooBar @foo="bar" />`, `<FooBar>...</FooBar>`, etc.

When a component is "invoked" in sub-expression form, we propose that it should
produce a new curried component with the given arguments. That is, if `foo-bar`
refers to a component value, then `(foo-bar)`, `(foo-bar "baz")` and
`(foo-bar bat="baz")` have the same semantics as `(component foo-bar)`,
`(component foo-bar "baz")` and `(component foo-bar bat="baz")`, respectively.

This allows curly invocations to work:

```hbs
{{foo-bar}}
{{foo-bar "baz"}}
{{foo-bar baz="bat"}}
```

In the first case, it refers to the component by value. Since the value happens
to be a component in this case, the rendering engine will invoke it at runtime.
The second and the third case relies on automatic parentheses insertion – they
desugars into `{{(foo-bar "baz")}}` and `{{(foo-bar baz="bat")}}`, respectively,
which produces anonymous curried components, which are also invoked at runtime.

If a component value, curried or not, is passed to an attribute position, it
will result in a runtime error as there is no sensible behavior there.

We propose to apply these same rules to modifiers as well – sub-expression
invocations will curry the arguments. If values other than (possibly curried)
modifiers are passed to a modifier position, it will result in a runtime error.
Conversely, if a modifier value is passed to content or attribute position, it
will also result in a runtime error.

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
