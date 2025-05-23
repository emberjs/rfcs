---
stage: accepted
start-date: 2025-05-23T00:00:00.000Z 
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - data
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1103 
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

# How Autotracking works 

## Summary

An explanation of the depths of the reactivity system we've been using and refining since Ember Octane (ember-source 3.13+).

_Required_ reading:
- [_How Autotracking Works_ by @pzuraq]https://www.pzuraq.com/blog/how-autotracking-works
- [_Reactivity Guides_ from glimmer-vm](https://github.com/glimmerjs/glimmer-vm/pull/1690/files#diff-ed98ef50f3453cdad3af6c62d003b3e212e945b5334b507be2ae3a05fbc7b804) 


> [!NOTE]
> These are all internal details, and we will need to be changing some of how this works over time to make improvements to rendering.

## Motivation

Unlike other RFCs, this RFC is more about documenting the details and concepts of reactivity at a low level so that we may build other primitive utilities that build on the concepts laid out in this document. This unblocks implementation of [`Cell`](https://github.com/emberjs/rfcs/pull/1071), Resources, and other primitives, and gives us more details to work with when describing reactive behavior.

This is just an overview, and much of it is copied from the source-of-truth, the markdown docs code-located with the sourcecode that implements these behaviors.

Details and further updates are not and will not be added to this document -- it's all in the realm of private API.

## Detailed design

### Walkthrough: setting a value

Given:
```gjs
import { tracked } from '@glimmer/tracking';

class ModuleState {
    @tracked count = 0;

    increment() {
        this.count++;
    }
}

const state = new ModuleState();

<template>
    <output>{{ state.count }}</output>
    <button {{on "click" state.increment>Increment</button>
</template>
```

And we 
1. observe a render, 
2. and then click the button,
   - and then observe the output count update.

How does it work?

There are a few systems at play for autotracking:
- [tags][^vm-tags]
- [global context][^ember-global-context]
- the environment / delegate
- some [glue code][^ember-renderer] that [configures][^ember-renderer-revalidate] the [timing specifics][^ember-renderer-render-transaction] of when to [render updates][^ember-renderer-render-roots]


[^vm-tags]: https://github.com/glimmerjs/glimmer-vm/blob/d86274816a21c61fbc82059006fe7687ca17dc7e/packages/%40glimmer/validator/lib/validators.ts#L1
[^ember-global-context]: https://github.com/emberjs/ember.js/blob/132b66a768a9cabd461908682ef331f35637d5e9/packages/%40ember/-internals/glimmer/lib/environment.ts#L21
[^ember-renderer]: https://github.com/emberjs/ember.js/blob/132b66a768a9cabd461908682ef331f35637d5e9/packages/%40ember/-internals/glimmer/lib/renderer.ts#L613C1-L614C1
[^ember-renderer-revalidate]: https://github.com/emberjs/ember.js/blob/132b66a768a9cabd461908682ef331f35637d5e9/packages/%40ember/-internals/glimmer/lib/renderer.ts#L626
[^ember-renderer-render-transaction]: https://github.com/emberjs/ember.js/blob/132b66a768a9cabd461908682ef331f35637d5e9/packages/%40ember/-internals/glimmer/lib/renderer.ts#L573
[^ember-renderer-render-roots]: https://github.com/emberjs/ember.js/blob/132b66a768a9cabd461908682ef331f35637d5e9/packages/%40ember/-internals/glimmer/lib/renderer.ts#L524

#### 1. leading up to observing a render

```
renderSync
read: count
```

#### 2. click the button
```
# increment()
read: count
set: count
# scheduleRevalidate()
env.begin
env.rerender
read: count
env.commit
```

the output, `count` is rendered as `1`


### A minimal renderer

[JSBin, here](https://jsbin.com/mobupuh/edit?html,output)

> [!CAUTION]
> This is heavy in boilerplate, and mostly private API. This 300 line *minimal* example, should be considered our todo list, as having all this required to render a tiny component is _too much_.


## How we teach this

n/a

## Drawbacks

We're still exploring what drawbacks are of our approach. We need to remove a lot of legacy code and support (or extract it to opt-in techniques (packages, lazy opcodes, etc) to really see how our rendering and reactivity approach will truely compare to our peer frameworks who don't have the support that we guarantee for users of Ember.

## Alternatives

n/a

## Unresolved questions

n/a
