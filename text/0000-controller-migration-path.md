- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Migration away from controllers

## Summary

 - Provide alternative APIs that encompass existing behavior currently implemented by controllers.
   1. query params
   2. passing state and handling actions to/from the template

 - Out of scope:
   - Deprecating and eliminating controllers -- but the goal of this RFC is to provide a path to remove controllers from Ember altogether. 

 - Not Proposing
   - Adding the ability to access more properties than `model` from the route template
   - Adding the ability to invoke actions on the route.
   - Any changes to the `Route`
   
This'll explore how controllers are used and how viable alternatives can be for migrating away from controllers so that they may eventually be removed.

## Motivation

 - Many were excited about [Routable Components](https://github.com/ef4/rfcs/blob/routeable-components/active/0000-routeable-components.md).
 - Many agree that controllers are awkward, and should be considered for removal (as mentioned in many #EmberJS2019 Roadmap blogposts).
 - There is, without a doubt, some confusion with respect to controllers:
   - should actions live on controllers?
   - how are they different from components?
   - controllers may feel like a route-blessed component (hence the prior excitement over the Routable-Components RFC)
   - controllers aren't created by default during `ember g route my-route`, so there is a learning curve around the fact that route templates can't access things on the route, and must create another new file, called a controller.
 - For handling actions from the UI, component-local actions and actions that live on services are a better long-term maintenance pattern. Dependency injection especially from the store and router services for fetching / updating data as well as contextually redirecting to different routes. 
 - State and Actions on the controller encourage prop drilling where arguments are passed through many layers of components which make maintenance more difficult.


## Detailed design

First, what would motivate someone to use a Controller today? Borrowing from [this blog post](https://medium.com/ember-ish/what-are-controllers-used-for-in-ember-js-cba712bf3b3e) by [Jen Weber](https://twitter.com/jwwweber): 

1. You need to use query parameters in the URL that the user sees (such as filters that you use for display or that you need to send with back end requests)  

2. There are actions or variables that you want to share with child components  

3. (2a) You have a computed property that depends on the results of the model hook, such as filtering a list

All of this behavior can be implemented elsewhere, and most of what controllers _can_ do is already more naturally covered by components.

### 1. Query Params

In this other RFC that [proposes adding a @queryParam decorator and service to manage query params](https://github.com/emberjs/rfcs/pull/380), query-param management would be pulled out of the controller entirely.

Presently, query params live on controllers, which are singletons, allowing query param values to be set, and unset, when navigation to and away from a route. Query params can have the same behavior implemented on a service, which can be tested out on an addon, [ember-query-params-service](https://github.com/NullVoxPopuli/ember-query-params-service).

Behavior like `replaceState` and `refreshModel: true`, would still live on the route as that behavior is more of a route concern than it is a concern of the value of the query params.

- **Mapping a Query Param to state**

  Old: query params _must_ come from a controller.
  ```ts
  import Controller from '@ember/controller';

  export default class SomeController extends Controller {
    queryParams = ['foo'];
    foo = null;
  }
  ```
  ```hbs
  <SomeComponent @foo={{this.foo}} />
  ```

  New: query params can be used anywhere via dependency injection.
  ```ts
  import Component from "@glimmer/component";
  import { queryParam } from "ember-query-params-service";

  export default class SomeComponent extends Component {
    @queryParam foo;

    addToFoo() {
      this.foo = (this.foo || 0) + 1;
    }
  }
  ```

- **Mapping a Query Param to a state of a different name**

  Old: controller must exist to define query params
  ```ts
  import Controller from '@ember/controller';

  export default class SomeController extends Controller {
    queryParams = {foo: 'bar' };
    bar = null;
  }
  ```

  New: No controller exists
  ```ts
  import Component from "@glimmer/component";
  import { queryParam } from "ember-query-params-service";

  export default class SomeComponent extends Component {
    @queryParam('foo') bar;

    addToFoo() {
      this.bar = (this.bar || 0) + 1;
    }
  }
  ```

### 2. Passing State and Handling Actions to/from the Template

In order to fully eliminate the controller, there _must_ be a way to handle computed properties or state along with responding to actions from the template. Components can cover this already, and would be an intuitive way to interact with the view and behavioral layer.

Currently, controllers + templates must be used in the app by having:
 - for classic apps:
   - `app/controllers/{my-route-name/or-path}.js`
   - `app/templates/{my-route-name/or-path}.hbs`
 - for pods apps:
   - `{pod-namespace}/{my-route-name/or-path}/controller.js`
   - `{pod-namespace}/{my-route-name/or-path}/template.hbs`
 - for MU or (future) MU-inspired layouts:
   - `src/ui/routes/{my-route-name/or-path}/controller.js`
   - `src/ui/routes/{my-route-name/or-path}/template.hbs`

The proposed new behavior would be that when a controller is not present, and there is only a template, that template becomes a template-only/stateless component. The template-only/stateless component has a number of advantages as it forces the developer to separate UI layout and to think about the semantic breakdown of the template. Here are some existing examples of template-only/stateless route templates in an app made today:

from emberclear.io
 - [routes/application/template.hbs](https://github.com/NullVoxPopuli/emberclear/blob/master/packages/frontend/src/ui/routes/application/template.hbs)
 - [routes/contacts/template.hbs](https://github.com/NullVoxPopuli/emberclear/blob/master/packages/frontend/src/ui/routes/contacts/template.hbs)

Sample:

```hbs
<section class='section'>
  <div class='container'>
    <Header @contacts={{this.model.contacts}} />

    <div class='content'>
      <ContactTable />
    </div>
  </div>
</section>
```

Everything is encapsulated in components so that intent is clear as to which part of the template is responsible for what behavior. However, this is only the simplest use case, where all derived state is handled by the components rendered by the existing template.

For the case where we _need_ derived state at the root of our route's tree.
 - for classic apps:
   - `app/components/{my-route-name/or-path}.js`
   - `app/templates/{my-route-name/or-path}.hbs`
 - for pods apps:
   - `{pod-namespace}/{my-route-name/or-path}/component.js`
   - `{pod-namespace}/{my-route-name/or-path}/template.hbs`
 - for classic apps after [RFC #481](https://github.com/emberjs/rfcs/pull/481) (Colocated Component and Template files)
   - `app/components/{my-route-name/or-path}.js`
   - `app/components/{my-route-name/or-path}.hbs`
 - for MU or (future) MU-inspired layouts:
   - `src/ui/routes/{my-route-name/or-path}/component.js`
   - `src/ui/routes/{my-route-name/or-path}/template.hbs`

For current classic apps, this may feel a bit like resolution magic, but it's just the same lookup rule as for controllers, just in a different folder. This changes makes more contextual sense with pods and module unification (or some future derivitive of module unification) where the backing context to the template is co-located alongside the template.

These are not "routable components", but components that are invoked after the `model` hook resolves in the route, recieving only a single property `@model`.

 - Before: with controllers
    ```ts
    import Route from '@ember/routing/route';

    export default class SomeRoute extends Route {
      async model() {
        const contacts = await this.store.findAll('contact');

        return { contacts };
      }
    }
    ```
    ```ts
    import Controller from '@ember/controller';

    export default class SomeController extends Controller {
      get onlineContacts() {
        return this.model.contacts.filter(contact => {
          return contact.onlineStatus === 'online';
        });
      }
    }
    ```
    ```hbs
    {{#each this.onlineContacts as |contact|}}
      {{contact.name}}
    {{/each}}
    ```

 - After: the controller is swapped for a component 
     ```ts
    import Route from '@ember/routing/route';

    export default class SomeRoute extends Route {
      async model() {
        const contacts = await this.store.findAll('contact');

        return { contacts };
      }
    }
    ```
    ```ts
    import Component from "@glimmer/component";

    export default class SomeComponent extends Component {
      get onlineContacts() {
        const { model } = this.args;
        
        return model.contacts.filter(contact => {
          return contact.onlineStatus === 'online';
        });
      }
    }
    ```
    ```hbs
    {{#each this.onlineContacts as |contact|}}
      {{contact.name}}
    {{/each}}
    ```

### Lookup Priority Rules

For each of these, assume that all parent routes/templates/etc have resolved.
Because there are 4 or 5 project structure layouts in flux, the notation here is what is used inside the resolver.

Given a visit to `/posts`
- lookup `template:posts`  
  - if a controller exists, the template will be rendered as it is today.  
  - if a controller does not exist, a template-only component will be assumed, passing the `@model` argument. This will make interop between controllers' templates and template-only component / used-for-route template components inconsistent, so controllers' templates should also get a `@model` argument.
  - if a component by the same name exists, it is ignored and must be invoked manually.
- lookup a component named "posts", `component:posts` and `template:components/posts`.   
  NOTE: if the `template:posts` template was found, this step does not happen.  
  - this component is invoked _as if_ `template:posts` was defined as `<Posts @model={{this.model}} />`
- default to `{{outlet}}` or `{{yield}}` for sub routes

## Drawbacks

It's very important that every use case for controllers today _can_ be implemented using the aforementioned techniques. If people are willing to share their controller scenarios, we can provide a library of examples of translation so that others may migrate more quickly.

It may be possible to write a codemod to help with the migration as the biggest difference, at leaste in this RFC's initial samples is that `model` is accessed on `this.args` instead of just `this`.  The tool and maybe even a linter rule could look for pre-existing components that have any of the same names as routes, and assert that they handle the `model` argument.


## Unresolved questions

 - What use cases of controllers are not covered here?

