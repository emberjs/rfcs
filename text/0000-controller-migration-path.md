- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Migration away from controllers

## Summary

_The goal is for components to make up the entirety of the view layer_.

 - Requires:
   - [RFC 496: Handlebars Strict Mode](https://github.com/emberjs/rfcs/pull/496)
   - [RFC 454: SFC & Template Import Primitives](https://github.com/emberjs/rfcs/pull/454)
   - [RFC 380: Add queryParams to the router service](https://github.com/emberjs/rfcs/pull/380)

 - Provide alternative APIs that encompass existing behavior currently implemented by controllers.
   1. query params
   2. passing state and handling actions to/from the route, loading, and error templates

 - Out of scope:
   - Deprecating and eliminating controllers -- but the goal of this RFC is to provide a _path_ to remove controllers from Ember altogether. 

 - Not Proposing
   - Adding the ability to invoke actions on the route.
   - Any changes to the `Route` or `Controller` as they exist today
   
This'll explore how controllers are used and how viable alternatives can be for migrating away from controllers so that they may eventually be removed.

## Motivation

 - Many were excited about [Routable Components](https://github.com/ef4/rfcs/blob/routeable-components/active/0000-routeable-components.md) (This RFC is _not_ "Routable Components").
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

See this RFC for [adding query params to the router service](https://github.com/emberjs/rfcs/pull/380), query-param management would be pulled out of the controller entirely.

### 2. Passing State and Handling Actions to/from the Template

In order to fully eliminate the controller, there _must_ be a way to handle computed properties or state along with responding to actions from the template. Components can cover this already, and would be an intuitive way to interact with the view and behavioral layer.

Currently, controllers + templates must be used in the app by having:
  - `app/controllers/{my-route-name/or-path}.js`
  - `app/templates/{my-route-name/or-path}.hbs`

The new behavior would allow components to be imported from anywhere in the app, and assigned to static properties on the Route.

```ts
import FooComponent, { LoadingComponent, ErrorComponent } from './components/Foo';
import Route from '@ember/routing/route';

export default class extends Route {
  static render = FooComponent;
  static loading = LoadingComponent;
  static error = ErrorComponent;

  async model() {
    const contacts = await this.store.findAll('contact');
    
    return { contacts };
  }
}
```

Everything is encapsulated in components so that intent is clear as to which part of the template is responsible for what behavior.
Additionally, all three properties may be set to the same component if it was desired to manage loading, error, and success states within components.

Depending on the state of the model hook, the different components will be rendered at have splatted arguments following this type signature:

```ts
enum State {
  Loading = "loading",
  Resolved = "resolved",
  Error = "error"
}

type RouteState<Data, E> =
  | { state: State.Loading }
  | { state: State.Resolved, data: Data }
  | { state: State.Error, error: E };
```

for non-typescript users, the values of splatted to the components will still adhere to the type signature.


This RFC proposes _three_ new static properties on the Route class as well the above mapping for arguments to the components for the static properties.

#### **render**

This is the component that will be rendered after the model hook resolves.
Today, there is a default template generated per-route that only contains `{{outlet}}`. In order to not break that behavior, the `render` property must not be required until both controllers and the route template are removed from the framework, after a deprecation RFC and the deprecation process).

If set, and there exists a classic route template, an error will be thrown saying that there is a conflict.

The render component will receive the `@state` argument as well as the `@data` argument, which represents the value of a resolved `model` hook.

#### **loading**

This is the component that will be rendered before the model hook has resolved, if present. 

If set, and there exists a classic loading template, an error will be thrown saying that there is a conflict.

The loading component only receives the `@state` argument.

#### **error**

This is the component that will be rendered if an error is thrown as they are thrown today.

If set, and there exists a classic error template, an error will be thrown saying that there is a conflict.

The error component will receive the `@state` argument as well as the `@error` argument, which will be the error object that is presently passed to the error action.


## Examples

Note that the _after_ paths are made up to just show that components are being imported from anywhere. Additionally, sample rendering in the template comments is for demonstrative purposes only.

- **a loading template related to the route**

  Given
  ```ts
  // app/router.js
  Router.map(function() {
    this.route('slow-model');
  });
  ```

  Before
  ```ts
  // app/routes/slow-model.js
  import Route from '@ember/routing/route';

  export default class extends Route {
    async model() {
      const slow = await this.store.findAll('slow-model');

      return { slow }
    }
  }
  ```
  ```hbs
  {{! app/routes/slow-model.hbs}}
  {{this.model.slow}} 

  {{! app/templates/slow-model-loading.hbs}}
  Loading...
  ```

  After
  ```ts
  // app/routes/slow-model.js
  import Route from '@ember/routing/route';
  import LoaderComponent from './components/loader';
  import ShowModelComponent from './components/show-the-model';

  export default class extends Route {
    static loading = LoaderComponent;
    static render = ShowModelComponent;

    async model() {
      const slow = await this.store.findAll('slow-model');

      return { slow }
    }
  }

  ```
  ```hbs
  {{! app/routes/components/show-the-model.hbs}}
  {{@state}}, {{@data.slow}} {{! renders: "resolved", <Array<SlowModel>> }}

  {{! app/routes/components/loader.hbs}}
  {{@state}} {{! renders: "loading"}}
  ```

- **an error template related to the route**

  Given
  ```ts
  // app/router.js
  Router.map(function() {
    this.route('erroring-model');
  });
  ```

  Before
  ```ts
  // app/routes/erroring-model.js
  import Route from '@ember/routing/route';

  export default class extends Route {
    async model() {
      throw new Error('whoops');
    }
  }
  ```
  ```hbs
  {{! app/routes/erroring-model.hbs}}
  {{this.model.slow}} {{! never renders}}

  {{! app/templates/slow-model-error.hbs}}
  {{! this.model is the error instance}}
  {{this.model.message}} {{! renders: "whoops"}}
  ```

  After
  ```ts
  // app/routes/slow-model.js
  import Route from '@ember/routing/route';
  import ErrorComponent from 'app-name/components/errors/route-error';
  import ShowModelComponent from './components/show-the-model';

  export default class extends Route {
    static error = LoaderComponent;
    static render = ShowModelComponent;

    async model() {
      throw new Error('whoops');
    }
  }

  ```
  ```hbs
  {{! app/routes/components/show-the-model.hbs}}
  {{@state}}, {{@data.slow}} {{! never renders }}

  {{! app/components/errors/route-error/template.hbs}}
  {{@state}} {{@error}} {{! renders: "error", <Error#{ message = 'whoops'}>}}
  ```

- **all async state to be managed in a component**

  Given
  ```ts
  // app/router.js
  Router.map(function() {
    this.route('slow-model');
  });
  ```

  Before
  ```ts
  // app/routes/slow-model.js
  import Route from '@ember/routing/route';

  export default class extends Route {
    async model() {
      const slow = await this.store.findAll('slow-model');

      return { slow }
    }
  }
  ```
  ```hbs
  {{! app/routes/slow-model.hbs}}
  {{this.model.slow}} 

  {{! app/templates/slow-model-loading.hbs}}
  Loading...

  {{! app/templates/slow-model-error.hbs}}
  {{this.model.error.message}}
  ```


  After
  ```ts
  // app/routes/slow-model.js
  import Route from '@ember/routing/route';
  import ShowModelComponent from './components/show-the-model';

  export default class extends Route {
    static error = ShowModelComponent;
    static render = ShowModelComponent;
    static loading = ShowModelComponent;

    async model() {
      throw new Error('whoops');
    }
  }

  ```
  ```hbs
  {{! app/routes/components/show-the-model.hbs}}
  {{#if (eq @state 'loading')}}
    {{@state}} {{! renders: "loading"}}
  {{else if (eq @state 'error')}}
    {{@state}} {{@error}} {{! renders: "error", <Error#{ message = 'timeout?'}>}}
  {{else}}
    {{@state}}, {{@data.slow}} {{! renders: "resolved", <Array<SlowModel>> }}
  {{/if}}
  ```

- **all async state to be managed in a component with backing class**
  ```ts
  // app/routes/slow-model.js
  import Route from '@ember/routing/route';
  import ShowModelComponent from './components/show-the-model';

  export default class extends Route {
    static error = ShowModelComponent;
    static render = ShowModelComponent;
    static loading = ShowModelComponent;

    async model() {
      throw new Error('whoops');
    }
  }

  // app/routes/components/show-the-model.js;
  import Component from '@glimmer/component';
  import { inject as service } from '@ember/service';
  
  export default class extends Component {
    @service router;
    @tracked readyYet = 'no';

    @action 
    onUpdate(/* element */) {
      switch (args.state) {
        case 'resolved':
          this.readyYet = 'yes';
          break;
        case 'loading':
          this.readyYet = 'loading!';
          break;
        case 'error':
          this.router.transitionTo('some-other-place');
          break;
        default:
          // can't actually get here
          break;
      }
    }
  }

  ```
  ```hbs
  <div {{did-update this.onUpdate}}>
    {{this.readeYet}}
  
    {{! app/routes/components/show-the-model.hbs}}
    {{#if (eq @state 'resolved')}}
      {{@state}}, {{@data.slow}} {{! renders: "resolved", <Array<SlowModel>> }}
    {{/if}}
  </div>
  ```

  The component signature in typescript could be something like this:

  ```ts
  import Component from '@glimmer/component';
  import { RouteState } from '@ember/routing/route';

  type Data = {
    user?: User;
    slow: SlowModel[]; 
  };

  type PossibleErrors = ClientError | NetworkError;
  type Args = RouteState<Data, PossibleErrors>;

  export default class extends Component<Args> {
    // ...
  }
  ```



## How we teach this

The RFCs, [496](https://github.com/emberjs/rfcs/pull/496) and [454](https://github.com/emberjs/rfcs/pull/454) are mainly required for the conceptual ideas that there can exist a world with no component resolver lookups and that components can be imported explicitly.

The current / classic / old ways of having the resolver _able_ to look up components isn't going to go away any time soon though. For backwards compatibility it most stay for a while. But as this'd be the recommended/explicit path forward, documentation/guides/etc will be updated to show how to use the aforementioned static properties as well as describing how the model hook is mapped to each of the components. 

More specifically recommended will be to use a separate component for each of the three states, but there should still be examples (as above) showing the one-component-does-everything approach.

In the release that makes these capabilities a default, any pages mentioning the controller, or the route template, or the loading/error templates should be removed or updated to fit the new way of using components


## Drawbacks

- Routes may feel like they are configuration (this may be fine)
