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


## Detailed design

### Query Params

In this other RFC that [proposes adding a @queryParam decorator and service to manage query params](https://github.com/emberjs/rfcs/pull/380), query-param management would be pulled out of the controller entirely.

### Error and Loading States


> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

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

However, there is a pattern that can be implement using React + React Router which would be good to have an equivelant of in Ember apps.

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

