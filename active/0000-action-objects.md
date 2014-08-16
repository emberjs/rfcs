- Start Date: 2014-08-15
- RFC PR: 
- Ember Issue: 

# Summary

I'd like to propose adding something similar to what I've been calling 
Action Objects to Ember. Action Objects are:

- Objects that wrap some functionality that can be invoked by calling
  their `.perform()` method (a la Command pattern).
- Promise aware: `.perform()` returns a `Promise.resolve()` that wraps
  the return value of `.perform()`
- Mini state machines that track and expose the resolution state of the promise returned
  from `.perform` in an easily bindable/queryable fashion
- Bound to the object they are declared on (similar to how present day
  `actions` methods are bound to the Route/Controller/Component they're
  declared on)

# Motivation

Promises are wonderful primitives for managing asynchrony, but building
the UI that reflects the state of a promise is cumbersome,
boilerplate-y, and error-prone. For example, if you're building a form
with a slow submit that POSTs to the server, you'll likely have to
manually:

- set some `isSubmitting` flag to true so that a) your submit button is
  grayed out during the request and b) so that you can check this flag
  to prevent multiple submits
- remember to set this flag to false in a `.finally()` (assuming you're
  wise enough to be using promises to begin with)
- set any error messages upon failure, and remember to blank them out
  next time you submit

Furthermore, if you'd like to use a stylized component as the submit
button, you have to forward all of that state to it in your template, 
when it'd really be way simpler if components had a way to fire actions
and track the success/failure state of those actions rather than a
strictly fire-and-forget.

Addons like [async-button](https://github.com/dockyard/ember-cli-async-button)
address this issue by firing actions with a callback arg that you can
optionally pass a promise to in order to notify the component of what
sort of pending/fulfillment/rejection state it is in, but knowing the
state of an action is extremely useful as a general concept and it
shouldn't be up to individual components to rewrite the boilerplate for
knowing what state an action/promise is in. 

Lastly, present day `actions` can be easily abused due to their
bubbling behavior; `actions` defined on `ApplicationRoute` are
essentially global handlers that can be triggered by otherwise totally
unrelated/disconnected components living anywhere in your app; while
there are some valid use cases for treating the Router as a state
machine with bubbling fire-and-forget mechanics, we've left the door
wide open for abuse in a majority of use cases. Action Objects on the
other hand are required to be explicitly passed around to any other
objects that need access to them (whether controllers, components,
templates, etc) which greatly limits the potential for abuse while
preserving present day conveniences/expectations as to what context an
action's perform function fires in (see below for more details).

# Detailed design

I've already implemented the basics; feel free to play around with the
following JSBin: http://emberjs.jsbin.com/ucanam/6070/edit

Action Objects (AO) are defined as top-level (i.e. not in an `actions` hash)
properties on an Ember Object, such as a Route, Controller, or
Component, via a computed property macro called `action`, which accepts
an optional argument which is the method that gets run when `.perform()`
is called on the AO. If no method is supplied, the method defaults to 
`Ember.K`, which is useful for defining stubbed actions on components
that are expected to be overridden by whoever is rendering the
component.

The `action` CP macro returns an `Action` object from its getter. This
`Action` object has many useful/bindable/queryable properties, such as:

    promise: null, // the promise returned from `perform()`
    pending:   false,
    resolved:  false,
    rejected:  false,
    fulfilled: false,
    state: 'default', // or 'pending', 'fulfilled', 'rejected'
    inactive: true, // when set to 'default'... could probably be renamed
    args: null,
    perform: function() {...}
    reset: function() {...} // resets the above state

The above values change predictably as the `perform()` promise
resolved/fulfills/rejects.

The value of `this` of the provided function is always the object that
the action has been declared on (similar to how present day methods in
the `actions` hash are bound to the Route/Controller they're declared
on).

# Drawbacks

One more Ember concept for folk to have to keep in their brains?

It's also annoying that you have to call `.perform()` on what is really
nothing more than a function, but I'm not sure we can preserve all the
niceties of Action Objects if they're just functions. Trek has suggested
it'd be nice to be able to start with functions that are easily passable
between controllers/routes, but then make it easy to upgrade them to
Aciton Objects when it becomes desirable to bind to / query an Action
Object's resolution state.

# Alternatives

I'm not certain of all the implications but we could possible just make
Action Objects normal functions with a bunch of additional state stored
on the function objects.

# Unresolved questions

I'd like to explore making actions dependent on the success/failure of
other objects, like dependent keys for actions; e.g. in a multi-step
process, it would be nice to prevent `step2` action from being
performable unless `step1` has fulfilled, and so in. 



