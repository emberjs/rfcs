---
stage: accepted
start-date: 2023-09-28T00:00:00.000Z # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/975
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

# Add context API

## Summary
Add a context API to allow sharing state with all descendants of a component
without prop-drilling.

## Motivation
Ember provides developers with great APIs for sharing state and props. Services
allow us to set global, application-wide state. Contextual components allow us
to expose components pre-filled with data.

However, the available solutions aren't enough for situations where we want to
access local state in deeply nested component trees. Services are global, but
what if we want to render two component trees side by side, and have each of
them expose a different "context state" to all descendants?

Similarly, while yielding does give us the ability to pass components or state
around, we'd still have to explicitly add arguments to all components rendered
within a section of the app.

Alternatively, we can also "prop-drill" - pass certain arguments through to all
components, sometimes just so that one deeply nested component can access the
information.

This is where a context API would greatly improve the developer experience. Some
examples of where context might be better than existing solutions.

- A design system

  In a component library, a `Card` component might accept a `backgroundColor`
argument. The same component could then also expose this background color in its
context state, making it available to all components nested inside it - no
matter how deep.

  Other library components, like buttons or text components, could then check what
background color they're rendered on, and adjust their own styles accordingly,
to make sure the text and background color contrasts are accessible, or just to
make sure the button background matches the card background ("danger"-style
buttons in "danger"-styled cards). These components could adjust their color no
matter how deeply nested they are inside the card, without us having to
explicitly pass the background color through multiple layers of components.

- Chart and graphical components

  In these scenarios there might be multiple layers being rendered, where yielding
or passing certain arguments can become cumbersome. A `Graph` component could
render an `Axis`, which would render `Tick`s and `TickLabel`s. The deeply-nested
components will need access to the root `Graph`'s config to modify what and how
they render.

  While passing the config through many layers is an option, context could make
this sort of code arguably easier to read, write and maintain.

The feature has also been requested in this past, for example:
- https://github.com/emberjs/rfcs/issues/775

## Detailed design
The details would still need to be worked out. The Glimmer VM already does a lot
of the tree-tracking background work that would be necessary to pass context
state through component layers.

For example:
- [DebugRenderTree](https://github.com/glimmerjs/glimmer-vm/blob/master/packages/%40glimmer/runtime/lib/debug-render-tree.ts)

  This class is used by Ember Inspector, and it already tracks the hierarchy of
components. We could do something similar to pass context state through
component layers as they render.

- [@glimmer/destroyable](https://github.com/glimmerjs/glimmer-vm/tree/master/packages/%40glimmer/destroyable)

  When elements render, the Glimmer VM [builds parent/child associations](https://github.com/glimmerjs/glimmer-vm/blob/68d371bdccb41bc239b8f70d832e956ce6c349d8/packages/%40glimmer/destroyable/index.ts#L100-L101)
in the destroyable module. Again, this is the sort of hierarchy tracking that
would be needed for a context API.


A very simple exploration into a `DebugRenderTree`-like context solution can be
found at https://github.com/customerio/glimmer-vm/tree/provide-consume-context.
This introduces a new class, which keeps track of "context provider" components,
and exposes their state to any descendant components.

The final implementation of context in Ember might be by using special
decorators, like:
```ts
@provide('my-context-name')
get value() {
  return 'my state';
}
```

which would register the property to be exposed as context, and which could be
consumed in any nested components via:

```ts
@consume('my-context-name') contextState;
```


## How we teach this


## Drawbacks


## Alternatives
Other popular frameworks already include context APIs:

- [React Context](https://react.dev/learn/passing-data-deeply-with-context)
- [Svelte getContext/setContext](https://svelte.dev/docs/svelte#setcontext)
- [Vue.js provide/inject](https://vuejs.org/guide/components/provide-inject.html)

There are also Ember addons to provide similar functionality in Ember:
- [ember-context](https://github.com/alexlafroscia/ember-context)
This addon introduces a context API that works quite well, but it doesn't
actually pass context through parent-child relationships in the render tree,
which makes it less robust and comes with caveats.

- [ember-contextual-services](https://github.com/NullVoxPopuli/ember-contextual-services)
This addon provides locally scoped services for routes only (not components).


## Unresolved questions
