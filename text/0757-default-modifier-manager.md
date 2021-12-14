---
Stage: Accepted
Start Date: 2021-05-17
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js, Learning
RFC PR: https://github.com/emberjs/rfcs/pull/757
---

# Default Modifier Manager

## Summary

Anything that can be in a template has its lifecycle managed by the manager pattern.
Today, we have known managers for `@glimmer/component`, `@ember/helper`, etc.
But what happens when the VM encounters an object for which there is no manager?,
such as a plain function? This RFC explores and proposes a default behavior for
those unknown scenarios when it comes to _modifiers_.

## Motivation

The addon, [ember-could-get-used-to-this](gh-ecgutt)
demonstrated that it's possible to (almost) use plain functions for modifiers.
And since Ember 3.25, modifiers can be passed around as values -- which has opened
a whole new world of ergonomics improvements where a developer could define a function in a
component class and use that function **as** a modifier. This is thanks to
ember-could-get-used-to-this implementing a Modifier Manager that knows what to do with plain functions.

This has the impact of greatly reducing the mental burden for app and addon devs, in that,
they no longer need to jump over to the app/addon modifiers directory to create a modifier
"just to do this one simple thing". In Ember 3.25+, developers can now do `<div {{this.myModifier}}>`.

The introduction of a plain-function modifier-manager is important because over the past recent years
(since the introduction of the Octane Edition), we've seen on numerous occasions,
folks repeatedly reaching for `{{did-insert}}`, `{{did-update}}`,
(from [`@ember/render-modifiers`](gh-render-modifiers)) and other lifecycle-esque utilities.

The README of [ember-modifier][gh-ember-modifier] does an excellent job of describing side-effects,
modifiers, how to think about them, and there overall philosophy.

There are two excerpts from the modifier addons' READMEs to keep in mind in the rest of this RFC,

> Conceptually, modifiers take tracked, derived state, and turn it into some sort of side effect - usually, mutating the DOM node they are applied to in some way, but they might also trigger other types of side effects.
>
> <cite style="display: block; text-align: right;">ember-modifier's README</site>

and

> In many cases, classic lifecycle hooks like didInsertElement can be rewritten as custom modifiers that internalize functionality manipulating or generating state from a DOM element. Other times, you may find that a modifier is not the right fit for that logic at all, in which case it's worth revisiting the design to find a better pattern.
>
> <cite style="display: block; text-align: right;">@ember/render-modifier's README</site>


An example:

```js
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';

export default class Example extends Component {
  @tracked numberOfClicks = 0

  // when the number of clicks is odd, add a JS movement effect / animation
  wiggly = (element) => {
    if (this.numberOfClicks % 2 === 0) {
      return;
    }

    // event callback
    let animate = () => {
      element.animate({ /* ... */ });
    }

    element.addEventListener('mousemove', animate);

    // Cleanup
    return () => element.removeEventListener('mousemove', animate);
  }

  increment = () => this.numberOfClicks++;
}
```
```hbs
Num clicks: {{this.numberOfClicks}}

<button
  type="button"
  {{this.wiggly}}
  {{on 'click' this.increment}}
>
  Try to click me
</button>
```

This demonstrates that a modifier can:
 - be defined as needed, there is no need for the modifier to live in its own file or a separate folder
 - semantically encapsulate setup and teardown logic -- the returned function is the teardown logic
 - have access to the element without leaking DOM details until the component class
 - react to changes to tracked data consumed within the setup stage of the modifier (`this.numberOfClicks`)

[gh-ember-modifier]: https://github.com/ember-modifier/ember-modifier/
[gh-render-modifiers]: https://github.com/emberjs/ember-render-modifiers
[gh-ecgutt]: https://github.com/tracked-tools/ember-could-get-used-to-this
[rfc-779]:https://github.com/emberjs/rfcs/pull/779

This pairs nicely with the [First-Class Component Templates RFC](rfc-779);
where modifiers may be defined in the same file as a template-only component.

<details><summary>Example</summary>

<!--
  If you're reading this RFC non-rendered, ignore the "jsx" language tag.
  "jsx" is incorrect, and isn't perfect, but "gjs" syntax highlighting
  hasn't been added to GitHub's highlighting yet.
-->

This could be an entire component

```jsx
const draggable = element => {
  element.addEventListener( /*... */ );

  return () => element.removeEventListener( /* ... */ );
};

<template>
  <div {{draggable}}>text content A</div>
  <div {{draggable}}>text content B</div>
  <div {{draggable}}>text content C</div>
  <div {{draggable}}>text content D</div>
</template>
```
_Note: `<template>` is experimental syntax, and should not be be the focus of this example_


</details>


## Detailed design

_A Default Manager is not something that can be chosen by the user, but is baked in to the framework
as a default so that a user doesn't have to build something to use a non-framework-specific variant
of the three constructs: Helpers, Modifiers, and Components._

The desired usage of a plain function in a template should be:
 - convenient
 - reduce boilerplate
 - be easily portable to JS for developers' mental model of how template and JS interact.

Which results in:
 - default to positional parameters
 - all named arguments are grouped into an "options object" as the last parameter.
   this happens to align with the _syntax_ of modifier invocation where named arguments may not appear
   before the last positional argument.

#### Example with mixed params

```hbs
<div {{this.myModifier 1 2 op="add"}}></div>
```
would be an example invocation of a function with the following signature
expressed as TypeScript for clarity:
```ts
type Options = { op: 'add' | 'subtract' }
class A {
  myModifier = (element: Element, first: number, second: number, options: Options) => {
    // ...
  }
}
```

#### Example with only positional parameters

```hbs
<div {{this.myModifier 1 2}}></div>
```
Because there are no named arguments passed in, the develope may opt to omit the options argument from the method signature.
```ts
class A {
  myModifier = (element: Element, first: number, second: number) => {
    // ..
  }
}
```
however, the options argument is still passed, but is equivalent to `{}`.


#### Example default modifier implementation

The type signature for any function-modifier could be:

```ts
import type { CapturedArguments as Arguments } from '@glimmer/interfaces';

type FnArgs<Args extends Arguments = Arguments> =
  | [Element, ...Args['positional'], Args['named']]
  | [Element, ...Args['positional'], {}];

type FunctionModifier<
  Args extends Arguments = Arguments,
  ModifierArgs extends FnArgs<Args> = FnArgs<Args>
> =
  | ((...args: ModifierArgs) => void) // setup, with no teardown
  | ((...args: ModifierArgs) => () => void) // setup with teardown

class A {
  myModifierA: FunctionModifier = (element, a, b, c, options) => {
    /* setup, with no teardown */
  }
  myModifierB: FunctionModifier = (element, a, b, c, options) => {
    /* setup */
    return () => { /* teardown */ }
  }
}
```


The details of the implementation of this modifier manager may take inspiration from
 - The [implementation](gh-vm-1348) for [RFC #756](rfc-756)
   (especially around where the default helper manager is chosen as the "fallback" when an existing manager is not found)
 - The modifier-manager in [ember-could-get-used-to-this](ecgutt-modifier)
   The main differences being that this RFC proposes
   - no need to import anything or wrap in a function call
   - instead of `(element, positionalArgs, namedArgs)` as the function signature, the signature
     will be `(element, ...positionalArgs, namedArgs)`

[gh-vm-1348]: https://github.com/glimmerjs/glimmer-vm/pull/1348
[rfc-756]: https://github.com/emberjs/rfcs/pull/756
[ecgutt-modifier]: https://github.com/tracked-tools/ember-could-get-used-to-this/blob/master/addon/-private/modifiers.js

## How we teach this

On the [Template Lifecycle, DOM, and Modifiers](tldm) page,
we'll want to insert a section on custom modifiers towards the end, and maybe swap out the
existing documentation referencing `ember-modifier` and/or `@ember/render-modifiers`.
We'll also want examples of how to manage modifiers with tracked data, maybe when to use a
local/private modifier, rather than something you'd put in `app/modifiers`.
Maybe a whole page is added to the guides on modifiers, their philosophy, when to use them, etc.

While the overall API for modifiers in general has been "not perfect", there has been a fair amount of
experimentation with modifiers since the 3.11 - 3.16 (pre-octane) era. Function modifiers are
significantly simpler than class-based modifiers, and the complexity of the specifics of class-
based modifiers is where most of the hesitance around adding modifiers by default has been.

Existing users of Ember should be made aware of these capabilities, once implemented, via
the release blog post, with some examples -- but folks watching developments in the ecosystem
will likely be aware of ember-could-get-used-to-this, which kind of implements
some parts of this RFC, so the migration path for users of ember-could-get-used-to-this should
be straight-forward.  That addon _may_ want to deprecate their similar functionality after
the proposed behavior here lands -- though, this RFC suggests implementation such that developers
may still use ember-could-get-used-to-this without disruption.

There will also be a polyfill that implements a modifier-manager for `Function.prototype`, like
the [helpers polyfill](helpers-polyfill) does for a helper-manager.

[tldm]: https://guides.emberjs.com/release/components/template-lifecycle-dom-and-modifiers/
[helpers-polyfill]: https://github.com/NullVoxPopuli/ember-functions-as-helper-polyfill

## Drawbacks


There could be some awkwardness around the last argument
passed to the function, as the type signature may not match how the call-site expects
the signature to be. Projects like [Glint](https://github.com/typed-ember/glint#readme)
would be essential for helping with clarity here.

## Alternatives

Class-based default modifiers would allow greater flexibility for creating modifiers, but would also
perpetuate the current problem of most surprise -- functions are often how folks start out thinking about problems.

## Unresolved questions

TBD
