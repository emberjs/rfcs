---
Stage: Accepted
Start Date: 
Release Date: Unreleased
Release Versions:
  ember.js: 
  glimmerjs: 
Relevant Team(s): ember.js, glimmerjs
RFC PR: https://github.com/emberjs/rfcs/pull/778
---

<!--- 
Directions for above: 

Stage: Leave as is
Start Date: Fill in with today's date, YYYY-MM-DD
Release Date: Leave as is
Release Versions: Leave as is
Relevant Team(s): Fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies
RFC PR: Fill this in with the URL for the Proposal RFC PR
-->

# Instrumentation using Debug-Render-Tree

## Summary

> Currently, instrumation for Ember component has payload with view data, which includes BOUNDS. This enables the render highlight feature for Ember-Inspector. We can also use it for tracking component rendering. Unfortunately, this is not supported by Glimmer component, the payload for Glimmer component only contains component name. However, debug-render-tree is fully supported by Glimmer component, they contains view data including BOUNDS. We can integrate instrumentation with debug-render-tree then we can send view data to Ember-Inspector. 

## Motivation

> First of all, Ember component supports such feature. The payload data can be used by some other apps. However, after Octane migration, this is no longer supported. It is an unexpected deprecation. I saw several requests for such feature to be continuously supported 
> https://discord.com/channels/480462759797063690/501388188762505216/684442737113563213
> https://discord.com/channels/480462759797063690/501388188762505216/686516553050751005
> Additionally, I recently added the component render highlight feature for Ember-Inspector. 
> https://drive.google.com/file/d/1jIUPaM69okID1tlroAFu8lead6zeKfi_/view?usp=sharing
> Other framework like ReactJS has such feature in dev-tool for several years. It is very important for developers, since we can then see when the component is updated or created. It not only help us improve the website performance by eliminating wasted rendering but also help detects potential bugs when component is destroyed and recreated. Without instrumenation with view data, these are impossible.


## Detailed design
### This is what we currently have:
> When a Glimmer component is rendered. "Bounds" data is send to debugRenderTree
> https://github.com/glimmerjs/glimmer-vm/blob/master/packages/@glimmer/runtime/lib/compiled/opcodes/component.ts#L841
```js
vm.env.debugRenderTree!.didRender(bucket, bounds);
```
> debugRenderTree save the data in the map `nodes`
> https://github.com/glimmerjs/glimmer-vm/blob/master/packages/@glimmer/runtime/lib/debug-render-tree.ts#L86
```js
  didRender(state: TBucket, bounds: Bounds): void {
    ...

    this.nodeFor(state).bounds = bounds;
    this.exit();
  }
```
> Currently, we only use the `capture` method to construct the render tree
> https://github.com/emberjs/ember.js/blob/master/packages/@ember/debug/lib/capture-render-tree.ts#L25
```js
export default function captureRenderTree(app: Owner): CapturedRenderNode[] {
  let renderer = expect(app.lookup<Renderer>('renderer:-dom'), `BUG: owner is missing renderer`);

  return renderer.debugRenderTree.capture();
}
```
### My proposal:
> We can add an array `newNodes` with `state` element. We push `state` to queue
```js
  didRender(state: TBucket, bounds: Bounds): void {
    ...

    this.nodeFor(state).bounds = bounds;
+   this.newNodes.push(state);
    ...
  }
```
> Then add a new method `takeNewNodes`, it returns an nested array of `node` mapped from `newNodes`, and reset `newNodes` to `[]` 
within EmberJS, when Glimmer component finish rendering, we call `takeNewNodes` and send the `Bounds` and `States` to instrumentation
> https://github.com/emberjs/ember.js/blob/master/packages/@ember/-internals/glimmer/lib/resolver.ts#L304


## How we teach this

> This change will be fully support component render highlight
> The `node` may have different shape of view data other then the payload from Ember component.  
> We need to update this doc. https://api.emberjs.com/ember/2.13/classes/Ember.Instrumentation

## Drawbacks

> There is no actual drawback I can think of. This change should NOT affect any performance and can be backward compatible.

## Alternatives

> Alternatively, we can call `_instrumentStart` as part of componentManager `update` method, similar to Ember component.
> https://github.com/glimmerjs/glimmer-vm/blob/master/packages/%40glimmer/manager/lib/public/component.ts#L168
> This is where Glimmer component calls `_instrumentStart`
> https://github.com/emberjs/ember.js/blob/master/packages/@ember/-internals/glimmer/lib/component-managers/curly.ts#L432
> It has several disadvantage.
 - Bonds data are generated as part of rendering, so starting point is in fact inaccurate. 
 - The `componentManager` for Glimmer component does not have access to node data. We need to make big changes to support it and the change could impact the performance.
 - In contract, another advantage of debugRenderTree is, it is enabled by ember-inspector. 
> https://github.com/emberjs/ember.js/blob/master/packages/@ember/-internals/environment/lib/env.ts#L111
> https://github.com/glimmerjs/glimmer-vm/blob/master/packages/@glimmer/runtime/lib/environment.ts#L122

## Unresolved questions

> We may not guarantee the component from current glimmer instrumentation is always mapped node data. We need test differen edge case.
> https://github.com/emberjs/ember.js/blob/master/packages/@ember/-internals/glimmer/lib/resolver.ts#L269
> We can create another method to manually removed the newNodes, `resetNewNodes` and call it before `_instrumentStart` is called.
> Additionally, It is unlikely, we need to check for any potental racing conditional. Is it possible we don't get all the node data before resolver is called? 
