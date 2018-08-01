- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Component Manager Bounds

## Summary

This is a follow up custom components RFC that outlines how we expose a component's DOM structure and associated semantics.

## Motivation
Historically, classic Ember components have had a built in escape hatch for doing DOM manipulation via `didInsertElement`. This allowed for components to encapsulate some 3rd party JavaScript library or do some fine grain DOM manipulation. We feel that is ability to drop down to a lower level abstraction makes components powerful.

Today, the custom components are not given direct access to the DOM structure; internally known as `Bounds`. This limitation means that custom components cannot easily encapsulate functionality in the same way that classic Ember components can. Expanding the capabilities of custom components to expose the `Bounds` is in order to increase the usefulness of the abstraction.

## Detailed design

In [RFC#2013](https://github.com/emberjs/rfcs/blob/master/text/0213-custom-components.md#detailed-design) we outlined the component lifecycle and how a custom component is associated with a manager. This design builds off off of that existing work and concepts.

### `elementHook` capability

The primary way of introducing new functionality to the component manager is through capabilities. To allow custom component managers to announce that they support element related hooks, we will introduce an `elementHook` flag to the capabilities. In doing so, bounds will be dispatched to a `didRenderLayout` method on the manager and `willDestroyLayout` will be called during destruction.

```js
import { capabilities } from '@ember/component';
import EmberObject from '@ember/object';

export default EmberObject.extend({
  capabilities: capabilities('3.5', {
    elementHook: true
  }),

  // ...
});
```

### `didRenderLayout` hook
Once the rendering engine has constructed the bounds for the component the `didRenderLayout` hook will be called with the `component` and a `Bounds` instance.


```js
import { capabilities } from '@ember/component';
import EmberObject from '@ember/object';

export default EmberObject.extend({
  capabilities: capabilities('3.5', {
    elementHook: true
  }),

  didRenderLayout(component, bounds) {
    component.bounds = bounds;
    component.didInsertElement();
  }
  // ...
});
```

#### Timing Semantics

`didRenderLayout` has the following timing semantics:

**Always**
- called **before** `didCreateComponent`
- called **after** the children's `didRenderLayout` hook is called
- called **after** DOM insertion

**May or May Not**
- be called in the same tick as DOM insertion
- be called in the same tick as `didCreateComponent`
- have the the parent's children fully initialized in DOM

In other words, components **should not** rely on the timing of DOM insertion outside of the component's own bounds.

### `willDestroyLayout` hook

To balance `didRenderLayout` we will expose a hook called `willDestroyLayout` that is invoked during destruction. It will receive the `component` instance.

```js
import { capabilities } from '@ember/component';
import EmberObject from '@ember/object';

export default EmberObject.extend({
  capabilities: capabilities('3.5', {
    elementHook: true
  }),

  didRenderLayout(component, bounds) {
    component.bounds = bounds;
    component.didInsertElement();
  },

  willDestroyLayout(component) {
    component.willDestroyElement();
  }
  // ...
});
```

#### Timing Semantics

`willDestroyLayout` has the following timing semantics:

**Always**
- called **before** `destroyComponent`
- called **after** the children's `willDestroyLayout` hook is called

**May or May Not**
- be called in the same tick as DOM removal
- be called in the same tick as `destroyComponent`

### `Bounds` instance

The `Bounds` instance that is passed to `didRenderLayout` has the following interface.

```ts

type FirstNode = Node;
type LastNode = Node;

interface Bounds {
  firstNode: Node | LastNode;
  lastNode: Node | FirstNode;
}
```

It's an important to note that the bounds is kept up to date with the structure of the template. For instance a template that looks like:

```hbs
<h1>Hello</h1>
```

Will have a `bounds` where `firstNode` and `lastNode` point to the same node.

```js
//... snip
didInsertElement() {
  console.assert(this.bounds.firstNode === this.bounds.lastNode, 'Node are the same'); // passes
}
```

In contrast a component with the following layout:

```hbs
<h1>Hello</h1>
<h2>Today is Friday!</h2>
```

Will have the `firstNode` point to the `h1` node and the `lastNode` will point to the `h2` node.

In cases where a component's layout doesn't have an `HTMLElement` at the top-level, but rather has some language construct, the bounds object is subject to mutate. For instance if your component's layout looks like the following.

```hbs
{{#if this.condtion}}<p>Truthy <button onclick={{action 'toggle'}}>Falsy</button></p>{{/if}}
```

If `this.condition` is truthy, `firstNode` and `lastNode` will point to the `p` node, however if the condition is toggled and the block is destroyed the bounds object is going to mutate. Instead of pointing at the `p` node, both `firstNode` and `lastNode` are going to point to a `comment` node. Because of this component authors should be defensive when it comes to coding against the bounds.


## How we teach this

What is proposed in this RFC is an extension to a low-level primitive. We do not expect most users to interact with this layer directly. Instead, most users will simply benefit from this feature by subclassing these special components provided by addons.

At present, the classic components APIs is still the primary, recommended path for almost all use cases. This is the API that we should teach new users, so we do not expect the guides need to be updated for this feature (at least not the components section).

For documentation purposes, each Ember.js release will only document the latest component manager API, along with the available optional capabilities for that realease. The documentation will also include the steps needed to upgrade from the previous version. Documentation for a specific version of the component manager API can be viewed from the versioned documentation site.

## Drawbacks

As noted in the detailed design, bounds is mutative object and may cause confusion.

## Alternatives

We could chose not to expose the DOM structure to custom components and spend time focusing on building out the user-facing APIs without rationalizing or exposing the underlying primitives.

## Unresolved questions

TBD?
