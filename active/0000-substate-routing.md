- Start Date: 2015-03-14
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Enable Ember developers to navigate to new routes that don't modify the URL of the page.

# Motivation

By not navigating to the route when the user transitions to it we add the functionality of rendering within outlets on the current page.

Allowing the route to control if the page transitions to a new location opens up the flexibility of the route having full control over how it renders itself. So as a company changes it's mind that a login form renders as a modal window, they can change all modal links quickly by changing the substate to be a page. 

By allowing the route to be configured to be a substate route, the location won't change.

Adding the ability for a route to load in an outlet when nested within a component sometimes means reconfiguring that components behaviour. This change would simplify the mixins or attributes a component expects.

# Detailed design

## Add substate definition

`Ember.Router.subState` will accept a route name and options, the only valid option would be `reservedPath` which would be for reserving a path to prevent any collisions of URLs in future.

Defining a route with the same path as the subState will result in an exception being fired on loading the application.

### Example
```
Router.map(function() {
  this.subState('login', {reservedPath: '/login'})
});
```

routes/login.js
```
export default Ember.Route.extend({
  renderTemplate: function(){
    this.render('login', {
      outlet: 'modal'
    });
  }
});
```

## Transition navigation

Just like any other routes in the page, a substate can be navigated to via the normal methods:
```
{{#link-to 'login' }}Login{{/link-to}}
```

```
this.transitionTo('login');
```

## Direct navigation

Directly loading the URL for the substate causes an exception even if the route has a URL specified. A route may choose to list the URL explicitly to reserve it for future usage.

## Migrating to a full route

Instead of using substates, the developer can change the code above to:
```
Router.map(function() {
  this.route('login', {path: '/login'})
});
```

# Drawbacks

Having substates that are not just loading screens and errors may lead to interaction heavy applications that are hard to debug because the state is held withing substates.

# Alternatives

## Property on a route
```
Router.map(function() {
  this.route('login', {path: '/login', subResource: true})
});
```
Less clarity may lead to developers missing the property.

## Route property
```
export default Ember.Route.extend({
  subResource: true,
  renderTemplate: function(){
    this.render('login', {
      outlet: 'modal'
    });
  }
});
```
This is working cross purposes by moving URL handling behaviour down into routes.

# Unresolved questions

- Consider how the router will implement not to transition URL
