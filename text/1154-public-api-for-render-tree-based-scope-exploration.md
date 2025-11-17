---
stage: accepted
start-date: 2025-11-17T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - data
  - framework
  - learning
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1154
project-link:
suite: 
---

<!--- 
Directions for above: 

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
suite: Leave as is
-->

<!-- Replace "RFC title" with the title of your RFC -->

# Public API for render-tree-based scope exploration 

## Summary

This RFC proposes a public API that enables safe experimentation with render-tree-based contextual data access patterns. The API provides scope-based synchronous access for all invokables (components, helpers, modifiers) across apps, engines, and addons.

## Motivation

The Ember community has long desired a Context API to share state across the render tree without prop drilling, but developers currently must resort to [abusing private Glimmer APIs](https://github.com/customerio/ember-provide-consume-context/blob/def6d34f639d56ebec1c7c8c888f86ec524b8688/ember-provide-consume-context/src/-private/override-glimmer-runtime-classes.ts#L1), creating upgrade risks and maintenance burdens. This RFC provides a public API that enables safe experimentation with Context patterns for use cases like theme systems, authentication state, and feature flags. The expected outcome is to establish a stable foundation for community exploration of contextual data access patterns while gathering real-world feedback to inform future official Context API design.

Additionally, this also enables exploration of providing _owner_ access to plain function helpers, modifiers, (all invokables).

## Detailed design

> [!NOTE]
> This RFC does not intend to tie is to any implementation details in our renderer, as experimentation in this area should be possible without changing existing behavior for pre-exitsing apps, and such renderer changes would not change the behavior of what this RFC proposes.

### The Public API

A few examples of usage, once implemented,

<details><summary>Accessing the owner in a plain function</summary>

```gjs
import { getScope } from '@ember/renderer';
import { getOwner } from '@ember/owner';

function currentRoute() {
  const scope = getScope();
  const owner = getOwner(scope); 

  return owner.lookup('service:router').currentRouteName;
}

<template>
  {{ (currentRoute) }}
</template>
```

</details>

<details><summary>Accessing the owner in a modifier</summary>

```gjs
import { getScope } from '@ember/renderer';
import { modifier } from 'ember-modifier';

const dimensions = modifier((element) => {
  const scope = getScope();
  const owner = getOwner(scope); 
  const resize = owner.lookup('service:resize');
  const callback => (entry) => {
    element.textContent = JSON.stringify(entry);
  };
  
  resize.observe(element, callback);

  return () => {
    resize.unobserve(element, callback)
  }
})
<template>
  <div {{dimensions}}></div>
</template>
```

</details>

<details><summary>Accessing via component getter</summary>

```gjs
import Component from '@glimmer/component';
import { getScope } from '@ember/renderer';
import { getOwner } from '@ember/owner';

export default class Demo extends Component {
  // equiv of @service router;
  get router() {
    return getOwner(getScope()).lookup('service:router');
  }

  <template>
    {{this.router.currentRouteName}}
  </template>
}
```

</details>

<details><summary>Writing to scope: A component adding to the scope for the component's descendants</summary>

```gjs
import { addToScope, getScope } from '@ember/renderer';
import Component from '@glimmer/component';

class MyCustomState { 
  /* ... */ 
  foo = 2;
}

class Demo extends Component {
  constructor(owner, args) {
    super(owner, args);

    let state = new MyCustomState(owner);
    addToScope(state);
  }

  <template>
    {{yield}}
  </template>
}

function accessIt() {
  let scope = getScope();
  let state = scope.entries.find(x => x instanceof MyCustomState);

  return state.foo;
}

// And then accessing:
<template>
  <Demo>
    {{ (accessIt) }} {{! prints 2 }}
  </Demo>
</template>
```

</details>

### Interface

```ts
import type Owner from '@ember/owner';

function getScope(): Scope | undefined;

interface Scope {
  /**
   * The owner can change based on rendering context,
   * renderComponent usage, engine usage, etc.
   * 
   * there are two ways this could be implemented:
   * - the framework could add the owner into 'entries' whenever it would change
   * - since the renderer _always_ knows the current owner, we reference that
   */
  get [OwnerSymbol](): Owner;

  /**
   * Each rendering layer will push into this array,
   * we can detect usage of `addToScope`, and pop when we exit rendering the thing that called `addToScope`.
   * 
   * This array should contain references *only* (but we can't enforce that).
   * For each invokable, we'll associate the scope with the invokable during its creation.
   * 
   * This means that modifications to the scope *cannot* occur during updates.
   */
  entries: unknown[];
}
```



### High-level "how it works"

Simular to the exiting debug-render-tree, we'd keep track of a set of "scope" objects throughout the render hierarchy.

And similar to how auto-tracking works, we'd enable access to the scope via:
```js
// psuedocode

// Evaluating an "Invokable" ( a curly expression, component, etc )
globalThis.scope = currentScope(); // implementation may depend on renderer strategy
invoke()
delete globalThis.scope;

// in @ember/renderer
export function getScope() {
  return globalThis.scope;
}
```

> [!NOTE]
> `globalThis.scope` is not literally being suggested here -- this variable should be private, and un findable by means other than the exported `getScope` function.

### How to add to the scope

### Limitations

Because this is a synchronous API, `getScope()` will be undefined after any `await`, and should not be used after an `await` as a result. A new rule to `eslint-plugin-ember` can help out here.


## How we teach this

This is a low-level API, and the API Docs generated from the JSDoc for these newly expored functions should have some examples.

It could also be worth demonstrating in the guides how to implement some use cases.

### Example service access without injection

```gjs
import { getScope } from '@ember/renderer';
import { getOwner } from '@ember/owner';

// in a hypothetical library 
export function service(name) {
  let scope = getScope();
  let owner = getOwner(scope);

  return owner.lookup(`service:${name}`);
}

/**
 * We no longer need class-based helpers to access services
 */
export function currentRouteName() {
  return service('router').currentRouteName;
}
```

Usage:
```gjs
<template>
  {{#let service "router") as |router|}}
    {{router.currentRouteName}}
  {{/let}}

  {{ (currentRouteName) }}
</template>
```

Both of these usages would be reactive as teh router changes routes.

### Example Context Implementation and Usage

```ts
// hypothetical ember-prototype-context library
import { addToScope, getScope } from '@ember/renderer';
import Component from '@glimmer/component';

export class Provide extends Component {
  constructor(owner, args) {
    assert(`Must pass @context`, args.context);
    assert(`Must pass a key`, args.key)

    provide([args.key, args.context]);
  }
}

export function provide(...args) {
  if (args.length === 3) {
    /**
     * TC39's Stage 1 Decorator 
     */
    return provideTC39Stage1Decorator(...args); // implementation omitted for brevity
  }

  if (args.length === 2 && 'kind' in args[1]) {
    if (args[1].kind === 'field') {
      /**
       * TC39 "Field" Decorator
       * 
       * @provide x = new State();
       */
      return (initialValue) => provide(initialValue.constructor, initialValue);
    }

    throw new Error(`Unsupported decorator usage for kind: ${args[1].kind}`);
  }

  /**
   *  provide(key, instance or state);
   */
  addToScope([key, context]);

  return context;
}

export function consume<ContextType>(key: ClassType<ContextType>): ContextType {
  let scope = getScope();

  // search up until we find our key
  for (let i = 0; i < scope.entries.length; i++) {
    let entry = scope.entries[i];

    if (!Array.isArray(entry)) continue;
    if (entry[0] === key) return entry[1];
  }
}

export class Consume extends Component {
  get context() {
    assert(`Must pass a key`, this.args.key);

    return consume(this.args.key);
  }

  <template>
    {{yield this.context}}
  </template>
}
```

This supports both class providing and consuming as well as template-based providing and consuming.

```gjs
import { Consume, Provide, consume, provide } from 'hypothetical ember-prototype-context library';

let id = 0;
class StateA { foo = 'a'; id = id++; }
class StateB { foo = 'b' }

const newA = () => new StateA();
const newB = () => new StateB();

class CustomProvide extends Component {
  @provide a = new StateA();
}

const CustomConsume = <template>
  {{yield (consume @key)}}
</template>;

<template>
  <Provide @key={{StateA}} @context={{ (newA) }}>
    <Consume @key={{StateA}} as |a|>
      {{a.foo}} === "a"
      {{a.id}} === 0
    </Consume>

    {{#let (consume StateA) as |a|}}
      {{a.foo}} === "a"
      {{a.id}} === 0;
    {{/let}}
  </Provide>

  <Provide @key={{StateA}} @context={{ (newA) }}>
    {{#let (provide StateB (newB)) as |b|}}
      {{b.foo}} === "b"

      <Consume @key={{StateB}} as |b|>
        {{b.foo}} === "b"

        <Consume @key={{StateA}} as |a|>
          {{a.foo}} === "a"
          {{a.id}} === 1
        </Consume>
      </Consume>
    {{/let}}

    <Consume @key={{StateA}} as |a|>
      {{a.foo}} === "a"
      {{a.id}} === 1
    </Consume>

    <CustomConsume @key={{StateA}} as |a|>
      {{a.foo}} === "a"
      {{a.id}} === 1;
    </CustomConsume>
  </Provide>


  <CustomProvide>
    {{#let (consume StateA) as |a|}}
      {{a.foo}} === "a"
      {{a.id}} === 2;
    {{/let}}
  </CustomProvide>

</template>
```


## Drawbacks

We already track a "stack" in our current renderer, but depending on how this is implemented, it could accidentally double the memory usage of that stack -- however most "scopes" will be empty / reused between render nodes, as what is in the scope is not expected to change that often.

Depending on implementation, there _could_ be a userland-induced memory leak, so we'll want to explore how to ensure the scope entries don't endlessly grow over time (we already manage this with the current renderer's stack (see second alternative below)).

Adding more APIs is always more for people to learn, but many frontend frameworks already have similar features, so developers should have an easy time picking this up if they wish (or the future features enabled by this API).

## Alternatives

- [ember-provide-consume-context](https://github.com/customerio/ember-provide-consume-context)
- [passing a third argument to component constructors that is the VM Stack](https://github.com/rtablada/ember-context-experiment/blob/main/app/components/UserName.gjs)

## Unresolved questions

n/a (so far)