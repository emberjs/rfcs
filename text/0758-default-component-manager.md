---
Stage: Accepted
Start Date: 2021-05-17
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js, Learning
RFC PR: https://github.com/emberjs/rfcs/pull/758
---

# Default Component Manager

## Summary

Anything that can be in a template has its lifecycle managed by the manager pattern.
Today, we have known managers for `@glimmer/component`, `@ember/helper`, etc.
But what happens when the VM encounters an object for which there is no manager?,
such as a plain function? This RFC explores and proposes a default behavior for
those unknown scenarios when it comes to _components_.

## Motivation

The addon, [ember-could-get-used-to-this](https://github.com/pzuraq/ember-could-get-used-to-this)
demonstrated that it's possible to use plain functions for helpers and modifiers.
Components on the other hand, have had no such "plain" / "vanilla" support / adoption from addons.
Maybe it's worth exploring what default behavior components should have.

This could reduce the number of dependencies folks need to worry about when building components.
Using native classes eliminates the reliance on a "framework thing" and further
lessens the appearance of "magic through object-oriented inheritance".

Additionally, a goal of baseclass-less-classes as components is to have 0 potential API
conflicts with userspace.

## Detailed design

Each implemented meta-manager needs a default manager specified. The logic for choosing when to use
a default manager is, at a high-level:

```
if (noExistingManagerFor(X)) {
  return defaultManager;
}
```
Where X is, for the purposes of this RFC, a Component.

_A Default Manager is not something that can be chosen by the user, but is baked in to the framework
as a default so that a user doesn't have to build something to use a non-framework-specific variant
of the three constructs: Helpers, Modifiers, and Components._


The default component manager could be similar to the component manager for `@glimmer/component`,
but be compatible with vanilla classes, such as:
```js
class MyComponent {
  @tracked count = 0;
}
```
Since the introduction of `@ember/destroyable`, we no longer _need_ a specific class for helping
with destruction.

If a component wanted to have a destruction hook, it could register one itself during construction:
```js
import { registerDestructor } from '@ember/destroyable';

export default MyComponent {
  constructor(...args) {
    super(...args);

    registerDestructor(this, () => this.myDestroy)
  }

  myDestroy() { /* ... */ }
}
```

and the component manager to support this could look like:

```js
import { setComponentManager } from '@ember/component';
import { destroy } from '@ember/destroyable';

class DefaultComponentManager {
  constructor(owner) { this.owner = owner; }

  createComponent(ComponentClass, args) {
    return new ComponentClass(this.owner, args);
  }

  getContext(component) {
    return component;
  }

  destroyComponent(component) {
    destroy(component);
  }
}

// side-effect -- this file needs to be imported for the component manager to be installed
setComponentManager((owner) => new DefaultComponentManager(owner), Object.constructor /* TBD */);
```

## How we teach this

The default component manager may not need its own guides-documentation, because the prevelance
of `@glimmer/component` is so widespread, documenting yet-another-way-to-define-a-component may
cause more harm than good.

Existing users of Ember should be made aware of these capabilities, once implemented, via
the release blog post, with some examples -- but folks watching developments in the ecosystem
will likely be aware of ember-could-get-used-to-this and ember-modifier, which both implement
some parts of this RFC, so the migration path for users of those addons should be straight-forward.
Those addons _may_ want to deprecate their similar functionality after the proposed behavior here
lands -- though, this RFC suggests implementation such that developers may still use both
addons without disruption.


## Drawbacks

People may begin to ask if they even need `@glimmer/component`, and that's
something that the core team may want to address -- some folks may feel like there is churn
in the component base-class, and don't know "what is correct", etc. The two approaches are
similar enough where we could introduce a migration via lint rule or codemod.

## Alternatives

The primary alternative to class-based components that folks are aware of is function based-components
(though this would force the introduction of new paradigms, e.g.: React's hooks)
  - React's hooks + ember's reactivity _may_ be something with exploring further and there have been a couple
    of experiments already:
    - https://github.com/rwjblue/ember-functional-component
    - https://github.com/lifeart/hooked-components
    - https://github.com/NullVoxPopuli/ember-function-component

## Unresolved questions

TBD
