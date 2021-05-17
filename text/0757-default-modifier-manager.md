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

The addon, [ember-could-get-used-to-this](https://github.com/pzuraq/ember-could-get-used-to-this)
demonstrated that it's possible to use plain functions for helpers and modifiers.
And since Ember 3.25, modifiers can be passed around as values -- which has opened
a whole new world of ergonomics improvements where a dev could define a function in a
component class and use that function **as** a modifier. This is thanks to
ember-could-get-used-to-this implementing a Modifier Manager that _knew what to do with plain functions.
This has the impact of greatly reducing helper burden for apps and addon devs, in that,
folks no longer need to jump over to the app/addon modifiers directory to create a modifier
"just to do this one simple thing". We can now do `<div {{this.myModifier}}>`.

This has the added benefit of, with template strict mode, using plain functions in module-scope
for helpers and modifiers.

For Example:

<!--
  If you're reading this RFC non-rendered, ignore the "jsx" language tag.
  "jsx" is incorrect, and isn't perfect, but "gjs" syntax highlighting
  hasn't been added to GitHub's highlighting yet.
-->

```jsx
const double = num => num * 2;
const resizable = element => { /* ... */ };

<template>
  {{double 2}}
  <div {{resizable}}></div>
</template>
```
_Note: `<template>` is experimental syntax, and should not be be the focus of this example_

Another aspect of which is exciting is that it becomes easier, in tests to grab
the output of yielding components:

```jsx
test('...', async function (assert) {
  let someElement;
  const captureElement = (element) => someElement = element;

  await render(hbs`
    <MyComponent {{captureElement}} />
  `);

  assert.equal(someElement.tagName, /* expected tag for this test */ );
  // ...
})
```

## Detailed design

Each implemented meta-manager needs a default manager specified. The logic for choosing when to use
a default manager is, at a high-level:

```
if (noExistingManagerFor(X)) {
  return defaultManager;
}
```
Where X is, for the purposes of this RFC, a Modifier.

_A Default Manager is not something that can be chosen by the user, but is baked in to the framework
as a default so that a user doesn't have to build something to use a non-framework-specific variant
of the three constructs: Helpers, Modifiers, and Components._

Modifiers' default manager should be similar to the
[Helpers' default](https://github.com/emberjs/rfcs/pull/756), in that they would be function-based.
The primary difference is that the first argument to a function-modifier is the attached `Element`,
and the function-modifier _may_ return a destruction function.

Example:
```js
class MyComponent {
  wiggle(element, first, second, options) {
    const doAnimation = { /* ... */ }

    element.addEventListener('touchstart', doAnimation);

    return () => element.removeEventListener('touchstart', doAnimation);
  }
}
```
```hbs
<button {{this.wiggle "first arg" "second arg"}}>
```
Similar to the helper manager, because the invocation of `{{this.wiggle ...` has no named arguments,
the last parameter in the `wiggle` function signature will be `undefined`.

The implementation of this modifier manager could look like:

```js
import {
  setModifierManager,
  capabilities as modifierCapabilities,
} from '@ember/modifier';
import { destroy, registerDestructor } from '@ember/destroyable';
import { setOwner } from '@ember/application';

class DefaultModifierManager {
  capabilities = modifierCapabilities('3.22');

  createModifier(fn, args) {
    return { fn, args, element: undefined, destructor: undefined };
  }

  installModifier(state, element) {
    state.element = element;
    this.setupModifier(state);
  }

  updateModifier(state) {
    this.destroyModifier(state);
    this.setupModifier(state);
  }

  setupModifier(state) {
    let { fn, args, element } = state;

    let argsForFn = args.positional;

    if (Object.keys(args.named).length > 0) {
      argsForFn.push(args.named);
    }

    state.destructor = fn(element, ...argsForFn);
  }

  destroyModifier(state) {
    if (typeof state.destructor === 'function') {
      state.destructor();
    }
  }
}

const DEFAULT_MODIFIER_MANAGER = new DefaultModifierManager();

// side-effect -- this file needs to be imported for the modifier manager to be installed
setModifierManager(() => DEFAULT_MODIFIER_MANAGER, Function.prototype);
```

## How we teach this

### Changes to Existing Pages

This RFC adopts a default modifier manager, which means that folks can use modifiers without
the assistance of a third-party addon, such as ember-modifier. This means that we'll benefit from
adding a page to the guides on modifiers, their philosophy, when to use them, etc. While the
overall API for modifiers in general has been "not perfect", there has been a fair amount of
experimentation with modifiers since the 3.11 - 3.16 (pre-octane) era. Function modifiers are
significantly simpler than class-based modifiers, and the complexity of the specifics of class-
based modifiers is where most of the hesitance around adding modifiers by default has been.

Existing users of Ember should be made aware of these capabilities, once implemented, via
the release blog post, with some examples -- but folks watching developments in the ecosystem
will likely be aware of ember-could-get-used-to-this and ember-modifier, which both implement
some parts of this RFC, so the migration path for users of those addons should be straight-forward.
Those addons _may_ want to deprecate their similar functionality after the proposed behavior here
lands -- though, this RFC suggests implementation such that developers may still use both
addons without disruption.


## Drawbacks


There could be some awkwardness around the last argument
passed to the function, as the type signature may not match how the call-site expects
the signature to be. Projects like [Glint](https://github.com/typed-ember/glint#readme)
would be essential for helping with clarity here.

## Alternatives

We could have class-based modifiers by default, but these could be less-easy to "whip up" on a whim.

## Unresolved questions

TBD
