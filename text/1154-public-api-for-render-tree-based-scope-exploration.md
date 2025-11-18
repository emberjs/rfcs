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

The Ember community has long desired a Context API to share state across the render tree without prop drilling -- and currently developers must resort to [abusing private Glimmer APIs](https://github.com/customerio/ember-provide-consume-context/blob/def6d34f639d56ebec1c7c8c888f86ec524b8688/ember-provide-consume-context/src/-private/override-glimmer-runtime-classes.ts#L1), creating upgrade risks and [maintenance burdens](https://github.com/customerio/ember-provide-consume-context/pull/49). The last action item for the [Context RFC, #975](https://github.com/emberjs/rfcs/pull/975) is to propose a public API for experimentation.

This RFC does not propose a solution for Context directly.

This RFC provides a public API that enables safe experimentation with Context patterns as well as aligning with some feature design discussed in Discord as well as spec meetings (all public) - enabling exploration of providing _owner_ access to plain function helpers, modifiers, (all invokables).



## Detailed design

> [!NOTE]
> This RFC does not intend to tie us to any implementation details in our renderer, as experimentation in this area should be possible without changing existing behavior for pre-existing apps, and such renderer changes would not change the behavior of what this RFC proposes.

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

<details><summary>Usage with renderComponent</summary>

```gjs
import Component from '@glimmer/component';
import { getScope, renderComponent } from '@ember/renderer';
import { getOwner } from '@ember/owner';

class Demo extends Component {
  // equiv of @service router;
  get name() {
    let scope = getScope();
    let owner = getOwner(scope);
    return owner.lookup('service:router')?.currentRouteName ?? 'no router';
  }

  <template>
    {{this.name}}
  </template>
}

function customRender() {
  let element = document.createElement('div');

  renderComponent(Demo, {
    into: element,
    owner: {
      lookup(name) {
        // not implemented
        return;
      }
    }
  });

  return element;
}

<template>
  {{ (customRender) }} {{! renders "no router"}}

  <Demo /> {{! renders "index", (if the URL is "/") }}
</template>
```

</details>

### Interface

Once implemented, this would be the stable public API:

```ts
import type Owner from '@ember/owner';

export function getScope(): Scope | undefined;

interface Scope {
  /**
   * The owner can change based on rendering context,
   * renderComponent usage, engine usage, etc.
   * 
   * there are two ways this could be implemented:
   * - the framework could add the owner into 'entries' whenever it would change
   * - since the renderer _always_ knows the current owner, we reference that
   * 
   * This "OwnerSymbol" is a placeholder for the Symbol used by `setOwner`, 
   * which enables `getOwner(scope)`
   */
  get [OwnerSymbol](): Owner;

  /**
   * For iterating up the render tree.
   * 
   * This the data within should contain references *only* (but we can't enforce that).
   */
  entries: Iterator<unknown>;
}

export function addToScope(x: unknown);
```



### High-level "how it works"

Similar to the existing debug-render-tree, we'd keep track of a set of "scope" objects throughout the render hierarchy.

And similar to how auto-tracking works, we'd enable access to the scope via:
```js
// psuedocode
// for demonstration only, not actual code

// Evaluating an "Invokable" ( a curly expression, component, etc )
globalThis.scope = currentScope(); // implementation may depend on renderer strategy
invoke()
delete globalThis.scope;

// in @ember/renderer
export function getScope() {
  return globalThis.scope;
}

export function addToScope(x) {
  let scope = getScope();

  assert('cannot add to scope when there is no current scope', scope);
  
  // Implementation omitted for brevity
  setScopeData(scope, x);
}
```

> [!NOTE]
> `globalThis.scope` is not literally being suggested here -- this variable should be private, and un findable by means other than the exported `getScope` function.


In renderer code, we already know the render tree, but none of it is exposed to users (it is all private API, and the debugRenderTree uses this for inspector support).  Each rendering node will have the opportunity to set some userland metadata via `addToScope`. This userland metadata is privately stored on the rendering node, and can only be accessed via `getScope`. 

For crawling up the userland metadata of the render tree, you'd iterate over the [entries Iterator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Iterator/Iterator) looking for whatever you need.

### What's in `scope.entries`?

- At a minimum, the `scope.entries` will contain a _root metadata_, the root of the application (containing the `owner`).
- Iterating `scope.entries` will always have _one_ iteration, unless `addToScope` is called during rendering.
- `scope.entries` is lazy, in that when rendering, we don't eagerly calculate what can be found within, nor while iterating (unless iteration completes, and hits the _root metadata_)
- For each render node, the metadata is undefined until set, so that iteration can skip over empty metadatas
- Changes to this `scope.entries` are _not_ reactive, because reactivity is not needed. The access of `scope.entries` is always just-in-time, because access-order is the same as render-order, and cleanup / "removal" happens in-tandem with render node cleanup (which only happens when all render children are cleaned up).

### Inspector

Because the inspector uses the debugRenderTree, and the debugRenderTree has access to everything that can have a scope -- the inspector can display the scope in the object inspector.

### Testing

Testing with scope is the same as using in the application or library code.

If you want to use a separate value or implementation of something higher up in the render tree, the test rendered code should be wrapped wiht a test implementation which sets the test-specific scope value.

### Limitations

Because this is a synchronous API, `getScope()` will be undefined after any `await`, and should not be used after an `await` as a result. A new rule to `eslint-plugin-ember` can help out here.


## How we teach this

This is a low-level API, and the API Docs generated from the JSDoc for these newly exported functions should have some examples.

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

Both of these usages would be reactive as the router changes routes.

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
  for (let entry of scope.entries) {
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

This isn't _Context_, and _Context_ is what our users want, but I think we want strong public APIs so that we can explore userland APIs, and then RFC what those end up being, what the requirements are, and then add that to the framework once the idea and capabilities is stable.

Adding more APIs is always more for people to learn, but many frontend frameworks already have similar features, so developers should have an easy time picking this up if they wish (or the future features enabled by this API).

## Alternatives

None.

### Prior Art

Context Explorations

- [ember-provide-consume-context](https://github.com/customerio/ember-provide-consume-context)
  - proposed in [RFC#975](https://github.com/emberjs/rfcs/pull/975)
    - only allows use in components
    - relies on tight coupling to the VM's current implementation
- [experiment passing the component tree to component managers](https://github.com/rtablada/ember-context-experiment/blob/main/app/components/UserName.gjs)
  - for this RFC, we don't want to change how component managers work, because this scope feature should work for _all_ invokables and _all_ `{{}}` regions
  - an RFC about this approach was opened at [RFC#1155](https://github.com/emberjs/rfcs/pull/1155)
    - This RFC focuses on changing component manager APIs (additively), which means that only components can interact with the context.
    - It exposes some currently internal private values for existing internal component managers, which are not meant for extension (both for safety and maintenance reasons).

  

Service-like things with non-string keys:

- [createStore](https://ember-primitives.pages.dev/6-utils/createStore.md) 
- [ember-polaris-service](https://github.com/chancancode/ember-polaris-service)

## Unresolved questions

- other names for `entries`? `hierarchy`? `ancestry`?
- add `.owner` property/getter on the `scope`, instead of requiring `getOwner(scope)`
- should `scope` be an object that you _must_ set key-value pairs on? (right now it's just a single value that can be anything)
