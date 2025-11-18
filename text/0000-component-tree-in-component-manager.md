---
stage: accepted
start-date: 2025-11-17 
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - framework
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
suite: 
---

# Add Component Path Traversal to Component Manager API

## Summary

This RFC proposes extending the custom component manager by adding a third argument for the parent component tree path.
This allows for safe experimentation for contextual features based on component hierarchy and without leaking VM internals or private component manager implementation details.

A proposed implementation of this RFC through PNPM patches and an example context component manager can be found here: https://github.com/rtablada/ember-context-experiment/tree/main

## Motivation

Having a context API similar to React's `useContext` or Vue's `provide/inject` API has been a long requested feature for Ember core.
There are existing context implementations ([DOM Context](https://ember-primitives.pages.dev/6-utils/dom-context.md) and [`ember-provide-consume-context`](https://github.com/customerio/ember-provide-consume-context)) in addon land.
However, these have their own caveats: extra DOM nodes, wrapping components, excess null checks/reactivity setup, and private API usage (in the case of `ember-provide-consume-context`).
This means it is hard to teach 1:1 for developers coming from other backgrounds and adds extra friction to implementing compatible interfaces for things like Radix which relies heavily on instantiation evaluation of context.

Primarily for the common use case when advising people using Ember and wanting context, there is often the request to be able to lookup context from component constructors (something not possible with either of the two above implementations).
This type of context lookup is often used in design systems or data provider components to allow for assertions, inherited default values, parent/child registration (if for some reason a component didn't want to use context APIs), and more.

Similarly, there is private API that must be used for Ember inspector and other developer/test tooling which rely on knowing or registering things based on the render tree stack as components are built.

## Detailed design

The custom component manager [RFC 213](./0213-custom-components.md) introduced a new API for extending the component initialization behavior for new components.
This RFC proposes to extend the `createComponent` argument signature to include an iterator for the parent component tree.
This is intentionally made as an iterator to allow for lazy evaluation of the component tree instead of a raw array which would add performance overhead as well as duplication of data in the array structure.

To implement this, `create` within the [Glimmer internal component manager implementation](https://github.com/glimmerjs/glimmer-vm/blob/8fb88bebfbd5acfafdf55e581e697e660008b890/packages/%40glimmer/manager/lib/public/component.ts#L141-L152) would be updated:

```ts
*getComponentTreeForStack(vmArgs) {
  for (let i = vmArgs.stack.stack.length - 1; i >= 0; i--) {
    const frame = vmArgs.stack.stack[i];

    if (frame?.state?.component) {
      yield frame.state.component;
    }
  }
}
create(owner, definition, vmArgs) {
  let delegate = this.getDelegateFor(owner),
    args = argsProxyFor(vmArgs.capture(), 'component'),
    stack = this.getComponentTreeForStack(vmArgs),
    component = delegate.createComponent(definition, args, stack);
  return new CustomComponentState(component, delegate, args);
}
```

This generator allows for lazy evaluation of the component tree, allowing component managers to only traverse as much of the tree as needed beginning at the nearest ancestor component.

No changes would be required for the produced GlimmerComponentManager, EmberGlimmerComponentManager, or EmberComponentManager implementations as they currently would ignore the stack argument (which means we also would not be adding any perf hit except for the iterator creation).

### Example Use - ContextComponent implementation


This could be then leveraged to make a context component addon/extension.
To start there would be a context container interface:

```ts
interface ContextContainer {
  constructor(
    key: unknown,
    value: unknown,
    parent?: ContextContainer
  );

  getContext(key: unknown): unknown;
}
```

This could be used to create a new ContextComponent for experimentation:

```ts
import Component, { EmberGlimmerComponentManager } from '@glimmer/component';
import { setComponentManager } from '@glimmer/manager';

export const CONTEXT_KEY = Symbol.for('ember-context');

export default class ContextComponent<T> extends Component<T> {
  constructor(owner: Owner, args: ArgsFor<T>, context: ContextContainer) {
    super(owner, args);
    this[CONTEXT_KEY] = context;
  }
}
```

In this API a providers would be declared using `Provide` components with the implementation:

```ts
export class ProvideContext extends ContextComponent {
  constructor(owner: Owner, args: ArgsFor<T>, context: ContextContainer) {
    super(owner, args, context);

    this[CONTEXT_KEY] = new ContextContainer(args.key, args.value, context);
  }
}
```

The component manager would then be implemented as:

```ts
function getLatestContextFromStack(components) {
  for (const component of components) {
    if (component instanceof ContextComponent) {
      return component[CONTEXT_KEY];
    }

    if (component instanceof ProvideContext) {
      return new ContextContainer(
        component.args.key,
        component.args.value,
        component[CONTEXT_KEY]
      );
    }
  }

  return new ContextContainer(null, null, null);
}

class ContextComponentManager extends EmberGlimmerComponentManager {
  createComponent(ComponentClass, args, stack) {
    const context = getLatestContextFromStack(stack);

    this.ARGS_SET.set(args.named, true);

    return new ComponentClass(this.owner, args.named, context);
  }
}

setComponentManager(
  (owner) => new ContextComponentManager(owner),
  ContextComponent
);
```

This implementation does not leverage any private API.

It should be noted that there is room for improvement in the above example to make it reactive via signals and to add a `Consume` component as well as a `@consume` decorator.

> [!NOTE]
> In making the example repo above there were two other small changes that I did need to make to implement my ContextComponentManager:
> 1. Expose ARGS_SET from BaseGlimmerComponentManager since this is required to be able to extend the BaseGlimmerComponentManager in debug builds without tripping assertions.
> 2. export EmberGlimmerComponentManager (and likely BaseComponentManager for completeness) from `@glimmer/component`. Without exporting these there is no way to provide a custom component manager for something that extends GlimmerComponent without breaking into private API.


## How we teach this

This would require an update to the inline API documentation for `setComponentManager` and `ComponentManager.createComponent` to reflect the new argument.
Since custom component managers are not commonly used, there would be no need to update guides or add excess documentation.

## Drawbacks

This does add an additional argument to the component manager API and while component instances are not private, there is possible concern that these instances could be misused.

## Alternatives

1. Add context as part of `owner`. There has already been work around decoupling `owner` from the application instance. Instead owner would be more of a weak proxy that eventually pointed up to the application instance.
  * Pro: there are other discussions around engines, renderComponent, etc where owner has been proposed to be more flexible and less tied to the application instance.
  * Con: owner is already a large API surface
  * Con: this may break existing assumptions about owner being tied to application instance
  * Con: this does not solve the problem of needing to traverse the component tree for context lookup 
2. Bake context into official VM
  * Pro: We ship context
  * Con: This does not allow for experimentation as any addons (like ember-provide-consume-context) would have to lock to internal VM implementations
  * Con: This would require more design and implementation work up front
3. [Scope RFC](https://github.com/emberjs/rfcs/pull/1154)
  * Pro: Allows for more than context lookup in MANY use cases
  * Pro: Allows for more reactive as well as lazy consume/provide patterns
  * Pro: Allows for context use in Helpers and Modifiers without API change
  * Con: Requires a much larger change to internals
  * Con: Does not give public API for things like dev tools that wish to inspect component tree

## Unresolved questions

2. Should this component tree also be added to helper and modifier managers?
