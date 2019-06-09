- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Migration away from controllers

## Summary

 - Provide alternative APIs that encompass existing behavior currently implemented by controllers.
 - Out of scope:
   - Deprecating and eliminating controllers -- but the goal of this RFC is to provide a path to remove controllers from Ember altogether. 
 - Not Proposing
   - Adding the ability to access more properties than `model` from the route template
   - Adding the ability to invoke actions on the route.
   
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

### Rendering

The templates used for routes should remain the same. Any context that would go in controllers can be split out to components. This has the advantage of components defining the semantic intent behind what they are rendering, such as in this example application template: [routes/application/template.hbs from emberclear](https://github.com/NullVoxPopuli/emberclear/blob/master/packages/frontend/src/ui/routes/application/template.hbs)
```hbs
<TopNav />

<NotificationContainer @position='top-right' />

<OffCanvasContainer class='no-overflow'>
  <UpdateChecker />

  {{outlet}}

</OffCanvasContainer>

<Modals />
<AppShellRemover />
```

This route-level template only defines the layout. Everything is encapsulated in components so that intent is clear as to which part of the template is responsible for what behavior.

In some cases, the route template may be as small as only having some wrapping styles ([routes/contacts/template.hbs from emberclear](https://github.com/NullVoxPopuli/emberclear/blob/master/packages/frontend/src/ui/routes/contacts/template.hbs))
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
Here, model is the only property that the template has access to. Because this template has a more narrow and specific purpose, it should be referred to explicitly as a "route template".

### Query Params

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

  Old:
  ```ts
  import Controller from '@ember/controller';

  export default class SomeController extends Controller {
    queryParams = ['foo'];
    foo = null;
  }
  ```

  New:
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

### Error and Loading States

According to the routing guides on [error and loading substates](https://guides.emberjs.com/release/routing/loading-and-error-substates/), for both errors and loading state, we need a template in various locations depending on our route structure and, for loading, which routes we think are slow enough to warrant a loading indicator (whether that be a spinner, or fake cards or other elements on the page). Additionally, there is a loading action fired per route that can optionally be customized in order to set parameters on the route's controller. 

Maybe something we could do to make things less configuration-based and also be more explicit is to set properties on routes. 

For example, maybe we want to set the error and loading UI on a particular route:
```ts
import Route from '@ember/routing/route';
import ErrorComponent from 'my-app/components/error-handler';
import LoadingComponent from 'my-app/components/loading-spinner';

export default class MyRoute extends Route {
  onError = ErrorComponent;
  onLoading = LoadingComponent;

  async model() {
    const myData = await this.store.findAll('slow-data');

    return { myData };
  }
}
```
This setting of components to use as the error and loading state render contexts allows us to more intuitively tie in to the dependency injection system as well as have very clear shared resources for this common behavior.

The actions that are called on error and loading could still exist, as those may be useful for maybe redirecting to different places in case of a 401 Unauthorized / 403 Forbidden error


Another scenario - maybe someone wants to use the same error and loading 
configuration for every route:

```ts
// utils/configured-route.js 
import Route from '@ember/routing/route';
import ErrorComponent from 'my-app/components/error-handler';
import LoadingComponent from 'my-app/components/loading-spinner';

export default class ConfiguredRoute extends Route {
  onError = ErrorComponent;
  onLoading = LoadingComponent;
}

// routes/posts.js
import Route from 'my-app/utils/configured-route';

export default class PostsRoute extends Route {
  async model() {
    const posts = await this.store.findAll('post');

    return { posts };
  }
}
```

This may not end up being a common pattern to have the same loading state of every route, but having at least the same configured Error handling per route could provide a way to display meaningful messages to end-users by default, rather than unexpected uncaught exceptions causing the UI to break unexpectedly.

The last scenario is using a one-off configuration, which would become more ergonomic whenever apps can move away from the 'classic' project structure.

Assuming apps will eventually get something similar to the module unification layout where one-off components can be co-located with routes:
```ts
import Route from '@ember/routing/route';
import ErrorComponent from './-components/error-handler';
import LoadingComponent from './-components/loading-spinner';

export default class MyRoute extends Route {
  onError = ErrorComponent;
  onLoading = LoadingComponent;

  async model() {
    const myData = await this.store.findAll('slow-data');

    return { myData };
  }
}
```
This would be most useful for patterns that use fake UI for loading, such as what facebook and linkedin do for their feeds.


Ideally, the onError component should catch _any_ error that occurs within the route's subtree. This should catch network errors that occur within the beforeModel, model, or afterModel hooks, and also during rendering. With component centric development, errors can happen anywhere, and it would be fantastic to have a centralized placed to handle those errors -- though this level of error handling may be outside the scope of this RFC.

#### Rendering the Error and Loading components

The loading component should only be rendered if the `onLoading` property is set and the model hook has not yet resolved. It does not need to receive any arguments. 

The error component should only be rendered if the `onError` property is set and an error occurs. It should receive an argument named `error` whos signature is any of, but not limited to, any of the subtypes of [`Error`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error). It'll be up to the application developer to handle how to specifically render the error. 

Example:

```ts
import Component from '@glimmer/component';
import { parseError } from 'app-name/utils/errors';

export default class ErrorComponent extends Component {
  get errorData() {
    const { error } = this.args;

    return parseError(error);
  }
}
```
```hbs
<div class='error-container'>
  <strong>{{error.title}}</strong>

  {{#if error.body}}
    <p>{{error.body}}</p>
  {{/if}}
</div>
```

```ts
// elsewhere, in app-name/utils/errors;
export function parseError(error: any): ParsedError {
  if (error instanceof ClientError) {
    const maybeJsonApiError = getFirstJSONAPIError(error);

    if (Object.keys(maybeJsonApiError).length > 0) {
      return { title: maybeJsonApiError.title, body: maybeJsonApiError.detail };
    }

    return {
      title: error.description,
      body: error.message,
    };
  }

  if (error instanceof NetworkError) {
    // parse something different for network errors
  }

  if (error instanceof SomeOtherError) {
    // parse something different unique to SomeOtherError
  }

  // All other errors
  const title = error.message || error;
  const body = undefined;

  return { title, body };
```


## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

These changes would require many changes to existing apps, and there likely won't be an ability to codemod people's code to the new way of doing things. This could greatly slow down the migration, or have people opt to not migrate at all. It's very important that every use case for controllers today _can_ be implemented using the aforementioned techniques. If people are willing to share their controller scenarios, we can provide a library of examples of translation so that others may migrate more quickly.

## Alternatives

To date (2019-06-09 at the time of writing this), there has been one roadmap blogpost to say to get rid of both routes and controllers and have everything be components, like React.  

React projects typically use dynamic routing which is only possible to build the full route tree through static analysis of a build tool that doesn't exist (yet?). Ember's Route pattern is immensely powerful, especially for managing the minimally required data for a route.

However, there is a pattern that can be implement using React + React Router which would be good to have an equivalent of in Ember apps.

The Route override + ErrorBoundary combo.

Consider the following 

```tsx
import { Route as ReactRouterRoute } from 'react-router-dom';
import ErrorBoundary from '~/ui/components/errors/error-boundary';

export function Route(passedProps) {
  const {...props } = passedProps;

  return (
    <ErrorBoundary>
      <ReactRouterRoute {...props} />
    </ErrorBoundary>
  );
}
```

where `ErrorBoundary` is defined as:
```ts
import React, { Component } from 'react';
import PageError from './errors/page';

// https://reactjs.org/docs/error-boundaries.html
export default class ErrorBoundary extends Componen {
  state = { hasError: false };
  // Update state so the next render will show the fallback UI.  
  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }
  // You can also log the error to an error reporting service
  componentDidCatch(error, info) { console.error(error, info); }

  render() {
    const { hasError, error } = this.state;

    if (hasError) return <PageError error={error} />;

    return this.props.children;
  }
}
```

what this allows is our apps to on a per-route basis be able to catch _all_ errors within a sub-route and perform some behavior local to that route -- provided that the entire app uses this custom `Route` and no longer uses the default one from `react-router-dom`. This is advantageous for a couple reasons:
 - whenever implementing a feature, bugfix, or whatever, anything wrong you do will be caught and displayed on the UI, rather than entirely breaking the UI or _causing infinite rendering loops_
 - Whenever there is an unexpected uncaught exception due to a yet-to-be-fixed bug, the UI for handling the error and displaying that error to the user is implemented _for free_. This helps with error reporting from users as the error will be rendered and there'd be no need to ask for a reproduction with the console open.

 Switching to totally dynamic routing would too big of a paradigm shift and it would remove one of the best features about Ember: the asyncronous state management for required data when entering a route. With dynamic routing, as in react-router, all of that responsibility would be pushed to the user -- every app would have a different way to handle loading and other intermediate state.

## Unresolved questions

 - What use cases of controllers are not covered here?

