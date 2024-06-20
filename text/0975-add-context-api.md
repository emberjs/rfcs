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

From a developer's perspective, services are currently the closest thing to context that exists in Ember.
The idea of registering a service on an application,
and then injecting it into any component to access its state,
is very similar to the idea of rendering a "context provider",
and then consuming its state in a descendant component.

In fact, services can be thought of as context keys provided on the application's scope,
which makes them available throughout the whole application.

Because of those conceptual similarities, and given the fact that Ember developers
are already familiar with service injections, it makes sense to keep the context API
identical to the services API.

There would be two new decorators for context:

A `@provide` decorator, which makes the value it decorates available to
the component's render tree:
```ts
@provide('my-context-name')
get value() {
  return 'my state';
}
```

And a `@consume` decorator, which retrieves the provided state. This is similar
to the `@service` decorator, and conceptually we may think of the context being
injected into the component:

```ts
@consume('my-context-name') contextState;
```

Because the `@service` decorator takes string names for services, the `@provide`
and `@consume` decorators would do the same. If, in future, the service API
changes to a different form, the context API should be changed at the same time,
to keep them identical.

### Testing
Testing utilities should be provided to make it easier to provide context
in render tests. A helper `provide` function could be used in tests to define
state for the components being tested.

This helper could be called in the `beforeEach` hook, to set up context on 
groups of tests, or in a single test to provide context for that test only.

In this example, `ThemedButton` consumes a theme context, and sets a class
depending on whether dark mode is enabled.
```js
import { module, test } from 'qunit';
import { setupRenderingTest } from 'ember-qunit';
import { render } from '@ember/test-helpers';
import { hbs } from 'ember-cli-htmlbars';
import { provide } from '@ember/context/test-support';

module('component tests', function (hooks) {
  setupRenderingTest(hooks);

  hooks.beforeEach(function (this) {
    provide(this, 'theme-context', {
      darkMode: true,
    });
  });

  test('it renders', async function (assert) {
    await render(hbs`
      <ThemedButton />
    `);

    assert.dom('button').hasClass('is-dark-mode');
  });

  test('it renders without dark mode', async function (assert) {
    provide('theme-context', {
      darkMode: false,
    });

    await render(hbs`
      <ThemedButton />
    `);

    assert.dom('button').doesNotHaveClass('is-dark-mode');
  });
});
```


## How we teach this
In the guides, context should be introduced after services and contextual
components, as it shares similarities with both topics. Understanding use cases
for context will be easier when developers are already familiar with the currently
existing state management patterns.

Context can be compared to services, but only being available within the 
component tree it's provided in, and with its lifecycle tied to the provider
component. The `@consume` decorator is very similar to `@service`, so Ember
developers will already be familiar with how to access context values this way.

We can build on concepts introduced in the contextual component docs to show
how context can be used to share state between components without having to
explicitly use yielded components.

The guides should provide thoughtful guidance on when to use context over services
or contextual components, and when it should be avoided.
For example, global state should still be managed with services. Or, providing
big context objects could lead to unnecessary rerenders, and some use cases
are better solved by more targeted args currying in contextual components.

As for teaching with examples, a `Form` component could be shown, providing a 
form state context.
Here, an Ember Data model is shared with nested components, which could be extended
to include form validation, for example.
```gjs
import Component from '@glimmer/component';
import { provide } from '@ember/context';

export default class Form extends Component {
  @provide('form-context')
  get formState() {
    return {
      model: this.args.model,
    };
  }

  <template>
    <form ...attributes>{{yield}}</form>
  </template>
}
```

A form input component could then consume this context to access the model:
```gjs
import Component from '@glimmer/component';
import { consume } from '@ember/context';

export default class FormInput extends Component {
  @consume('form-context') formState;

  get value() {
    return this.formState.model[this.args.name];
  }

  get errors() {
    return this.formState.model.errors[this.args.name];
  }

  <template>
    <input type="text" name={{@name}} value={{this.value}} class={{if errors "is-invalid"}} ...attributes />
    {{#each this.errors as |error|}}
      <div class="error">
        {{error.message}}
      </div>
    {{/each}}
  </template>
}
```

Whenever `FormInput` is rendered inside a `Form`, it would have access to the
context, without having to curry arguments like we do in contextual components.

The input could be used in another component like:
```gjs
import Component from '@glimmer/component';
import FormInput from './form-input';

export default class FormSection extends Component {
  <template>
    {{! Apply any styles or additional content }}
    <div ...attributes>
      <FormInput @name={{@name}} />
    </div>
  </template>
}
```

Which is then composed with the form:
```hbs
<Form @model={{this.model}}>
  <FormSection @name="firstName" />
  <FormSection @name="lasttName" />
</Form>
```

Library authors especially would benefit from this pattern, allowing them to
build more flexible component APIs.

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
