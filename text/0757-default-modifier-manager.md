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

By introducing a default modifier manager, we can enable standalone functions to be used as modifiers.

Today, when defining a modifier, developers need to both install _a_ library,
such as [ember-modifier](gh-ember-modifier) or [ember-could-get-used-to-this](gh-ecgutt), and wrap
a function an a utility from one of those libraries:

```js
// app/modifiers/scroll.js
import { modifier } from 'ember-modifier';

export default modifier(function scroll(el, _, { when: shouldScroll }) {
  if (shouldScroll) {
    el.scrollTop(0);
  }
});
```

With the default modifier manager proposed in this RFC,
we can write a function which receives the element and the arguments directly:

```js
// app/modifiers/scroll.js
export default function scroll(el, { when: shouldScroll }) {
  if (shouldScroll) {
    el.scrollTop(0);
  }
}
```

This complements the introduction of the [default-helper-manager](rfc-756) and is
especially nice when combined with strict mode templates
via [ember-template-imports](gh-ember-template-imports) (or as proposed in [First-Class Component Templates (RFC 779)](rfc-779))

[gh-ember-template-imports]: https://github.com/ember-template-imports/ember-template-imports

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
  // (this is the modifier definition)
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
as a default so that a user doesn't have to build framework-specific wrappers in order to use
modifiers, (or whatever the manager is managing -- helpers, etc).
This reduces the overall API surface area and boilerplate of the framework, allow developers
to lean more on general JavaScript.

The desired usage of a plain function in a template should be:
 - convenient
 - reduce boilerplate
 - be easily portable to JS for developers' mental model of how template and JS interact.

Which results in:
 - default to positional parameters
 - all named arguments are grouped into an "options object" as the last parameter.
   this happens to align with the _syntax_ of modifier invocation where named arguments may not appear
   before the last positional argument.

The following examples will use the example from the [Ember Guides](eg-3-27-on-click-outside), where a modifier
is used to implement a callback-invocation upon clicking _outside_ an element,
which could be useful for modals, menus, popovers, etc.

[eg-3-27-on-click-outside]: https://guides.emberjs.com/v3.27.0/components/template-lifecycle-dom-and-modifiers/#toc_out-of-component-modifications

For reference, this is the original implementation:

```js
// app/modifiers/on-click-outside.js
import { modifier } from "ember-modifier";

export default modifier((element, [callback]) => {
  function handleClick(event) {
    if (!element.contains(event.target)) {
      callback();
    }
  }

  document.addEventListener("click", handleClick);

  return () => {
    document.removeEventListener("click", handleClick);
  };
});
```


#### Example with mixed params

In this modal example, with the modal (a div) starting open, the
`this.onClickOutside` modifier is applied with mixed positional (`this.close`)
and named (`when`) arguments.

```hbs
{{#if this.isOpen}}
  <div {{this.onClickOutside this.close when=this.canLeave}}>
    modal contents
  </div>
{{/if}}
```
```js
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';

export default class Example extends Component {
  @tracked isOpen = true;
  @tracked canLeave = true;

  onClickOutside = (element, callback, options) => {
    function handleClick(event) {
      if (!options.canLeave) return;

      if (!element.contains(event.target)) {
        callback();
      }
    }

    document.addEventListener('click', handleClick);

    return () => {
      document.removeEventListener('click', handleClick);
    };
  }

  close = () => this.isOpen = false;
}
```

#### Example with only positional arguments

This example is the same as above, but with only positional arguments

```hbs
{{#if this.isOpen}}
  <div {{this.onClickOutside this.close this.canLeave}}>
    modal contents
  </div>
{{/if}}
```
```js
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';

export default class Example extends Component {
  @tracked isOpen = true;
  @tracked canLeave = true;

  onClickOutside = (element, callback, canLeave) => {
    function handleClick(event) {
      if (!canLeave) return;

      if (!element.contains(event.target)) {
        callback();
      }
    }

    document.addEventListener('click', handleClick);

    return () => {
      document.removeEventListener('click', handleClick);
    };
  }

  close = () => this.isOpen = false;
}
```

Because there are no named arguments passed in,
the developer may opt to omit the options argument from the method signature.
However, the options argument is still passed, but is equivalent to `{}`.


#### Example default modifier implementation

The type signature for any function-modifier could be:

```ts
import type { CapturedArguments as Arguments } from '@glimmer/interfaces';

type FnArgs<Args extends Arguments = Arguments> =
  | [Element, ...Args['positional'], Args['named']]
  | [Element, ...Args['positional'], {}]
  // For argument-less modifiers or
  // for modifiers defined in classes that consume tracked data
  | [Element];

// The return value may be a function, which is used for cleanup, but otherwise
// no return value is needed
type FunctionModifier<ModifierArgs extends FnArgs> =
  ((...args: ModifierArgs) => (void | () => void))

class A {
  myModifierA = (element, a, b, c, options) => {
    /* setup, with no teardown */
  }
  myModifierB = (element, a, b, c, options) => {
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
experimentation with modifiers since the 3.11 - 3.16 (pre-octane) era. Since Octane's release,
[ember-modifier](gh-ember-modifier) has had extensive usage and has proven that the design of
function-based modifiers is solid for its use-cases.
Function modifiers are significantly simpler than class-based modifiers,
and the complexity of the specifics of class-based modifiers is where most of
the hesitance around adding modifiers by default has been (around lifecycle management).

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
would be essential for helping with clarity here:
mostly to alert developers when modifier arguments do not match the defined type signature.
If one day Glint becomes a default that developers don't have to configure, regardless of the
usage of typescript or not, it can help protect people from easy mistakes, typos, etc.
(But this applies to a lot more than just modifier usage)

## Alternatives

Class-based default modifiers would allow greater flexibility for creating modifiers, but would also
perpetuate the current problem of most surprise -- functions are often how folks start out thinking about problems.

## Unresolved questions

TBD
