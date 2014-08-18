- Start Date: 2014-08-15
- RFC PR: 
- Ember Issue: 

# Summary

This RFC proposes a variety of improvements to Ember actions that I've
been testing out in my own apps to great success. I see two different
approaches for these improvements, one approach I'd call Action Objects,
another I'd call Action Functions, but they both fall under the umbrella
of what I'll just call Better Actions.

Better Actions:

- are promise-aware
- have bindable/queryable state (e.g. fulfilled, rejected, resolved)
- make it possible for action emitters (often Components) to capture the
  return value of an action
- do not automatically bubble and must be explicitly passed into whomever
  wants to call them

# Motivation

Present day Ember actions are largely influenced by a fire-and-forgot 
state machine semantics, which means:

- Actions can bubble anywhere from a controller all the way up to ApplicationRoute
- There are no return values to a fired action (only an exception if
  no one handles it)

In some use cases, this paradigm is appropriate; actions emitted from a
parent route might need to be handled different depending on which child
route is currently active, which is a use case well met by the
state-machiney-ness of route actions.

But most of the time, actions are just used to attach handlers to UI,
in which case the bubbling behavior is unused/unneeded, and can lead to
surprising regressions when an intermediate ancestor declares an action
with the same name as a pre-existing one. And in general, we've made it
all too easy to declare global handlers due to how all actions bubble to
ApplicationRoute.

In addition, it's often desirable for components to know what happened
to an action that it emitted:

- is the action still underway?
- did it fail?
- did it succeed?

Addons like [async-button](https://github.com/dockyard/ember-cli-async-button)
address this issue by firing actions with a callback arg that you can
optionally pass a promise to in order to notify the component of what
sort of pending/fulfillment/rejection state it is in, but knowing the
state of an action is extremely useful as a general concept and it
shouldn't be up to individual components to rewrite the boilerplate for
knowing what state an action/promise is in. 

If actions knew the state of the operations they wrapped and exposed
this state as queryable/bindable properties, many concise and clear
patterns for presenting/managing asynchrony are possible with a minimal
amount of boilerplate.

For example, consider a form that performs a slow POST to the server;
you'll need to

- prevent double submits
- provide UI feedback (e.g. "Submitting...") for all the async states
  the form submission can be in

And ideally you'd also like to be able to pass the state of the form
submission to a reusable component, such as a stylized submit button.

In present day Ember, implementing the above requires many flags and
other boilerplate state and is pretty complex and error-prone for such a
common use case.

If actions were promise-aware and exposed the state of the internal
promoise, we'd be able to write something like:

    // controller
    export Controller.extend({
      submitForm: action(function() {
        return ajax("/url", someData);
      })
    });

    // template
    <form {{action 'submitForm' on='submit'}}>
      {{#if submitForm.pending}}
        Please wait...
      {{else}}
        <input type="submit" value="Submit!">
      {{/if}}

      {{#if submitForm.rejected}}
        <p>
          Error: {{submitForm.rejected.value.message}}}
          {{! keep in mind that you could alias this in a controller
              to slim things down, but this would Just Work without
              lots of additional controller state}}
        </p>
      {{/if}}
    </form>

Note how all of this important state that the template needs to properly
display the various states a form could be in is entirely encapsulated
in the `submit` action. Without this, you have to clumsily do your own
promise-aware state management in `.then` `.catch` `.finally`. The
implementation of actions would by default prevent double submits if the
promise is currently in pending state, and at any point, the action's
state could be reset by calling `submitForm.reset()`.

And in the case that you had a stylized submit button component that you
wanted to reuse, you could pass it the submit action and the component
would be able to reason about the state of the action it was passed to
know how it should present its UI:

    // form template
    <form>
      ...
      {{async-button action=submitForm}}
      ...
    </form>

    // async-button layout template
    {{#if action.pending}}
      Please wait...
    {{else}}
      <button {{action action}}>Submit!</button>
    {{/if}}

This preserves the flexibility for both the outer form and async-button
component to respond to the state of the action. It's also very easy to
do `bind-attr class=":some-button action.state"` to make the various
promise states easily stylable.

I've been using this pattern exclusively (i.e. no `actions` hash) in two mobile Ember apps to
great success; it's cut down on much boilerplate, prevented
double-taps/submits, and has general encouraged a new paradigm where
actions are the originators and managers of 90% of 
controller/template state, and I've largely avoided
any additional flags that I have to remember to initialize / reset. I
still need to `reset()` actions, but everything after that, I get for
free. 

# Detailed design

I've experimented with two approaches to how to implement Better
Actions:

- Action Objects: operations are wrapped in an Ember.Object subclass
  that exposes the various promise states and has a `.perform()` method
  to invoke the operation. Spike implementation:
  http://emberjs.jsbin.com/ucanam/6070/edit
- Action Functions: actions are just functions with additional promise
  state set directly on them. The action function wraps the internal
  function you pass to it and handles calling the internal function with
  the correct context (the Route/Controller/Component that the action is
  defined on).
  Spike implementation: http://emberjs.jsbin.com/ucanam/6077/edit

If possible, I'd love to make Action Functions work because it means
that they could be invoked from JavaScript code via `this.someAction()`
rather than `this.get('someAction').perform()`. It would also allow for
an easier promotion from a plain ol JavaScript to an Action Function
since consumers wouldn't have to suddenly switch to calling `.perform()`
instead of just calling it like a normal function. 

But there are some issues with Action Functions:

1. We would need to change `ember-metal` to 
  [allow ChainWatchers to install on functions](https://github.com/emberjs/ember.js/blob/master/packages/ember-metal/lib/chains.js#L32)
2. We need to decide where and how to support some form of `.bind()`.

The latter issue is tricky; we want action functions to be declarable on
a parent object (e.g. a route or controller) and easily passable to some
child object to invoke (e.g. a controller or component) -- recall that
we are purposefully eschewing bubbling behavior, so we need to make sure
that when we pass Action Functions around, they still run their internal
function in the context of the parent object. 

Assuming we don't provide any sugar for automatically passing route
actions to controllers, it's tempting to try and make something like
this work:

    export Route.extend({
      someRouteAction: action(function() {
        // ...
      }),

      setupController: function(controller) {
        controller.set({
          controllerActionName: this.someRouteAction.bind(this)
        })
      }
    });

The problem is that if we use native `Function.bind` (which we can't
really do if we wanna support old browsers), it'll return a vanilla
function that doesn't have any of the Action Function API (promise state
and what not). Even if we override `.bind` in the Action Function API to
return whatever we want, we'd still need to return a separate
Action Function which is properly bound, and I can't really think of a
clean way without lots of ugly proxy magic to preserve the bindings to
the singleton, origin Action Function.

It seems then that if we want to support Better Actions just being
enhanced functions, we'd need to 

- Detect Action Functions at init time and auto-bind them to the object
  (fwiw, React does this)
- Override `.bind()` to no-op and/or emit a warning/error that
  Action Functions are already perma-bound to the object they're
  declared on.

Note that while this proposal doesn't intend to solve the issue of
components having to proxy forward an action through multiple layers of
child components, it does make it much simpler to forward an action,
since all that's involved is just passing the action object property
to child components from within the parent component's layout template,
rather than have to translate through many layers of intermediate 
old school action handlers and `sendAction()`s. You can see an example
of this [here](http://emberjs.jsbin.com/ucanam/6077/edit).

# Drawbacks

- Conflicts with existing Ember `actions` concept (maybe solvable by not
  describing them as actions, but rather just better functions?)
- API bloat?
- _Some_ of these issues could conceivably be handled by URL-less
  states (but I think this is a way better general primitive to offer
  that doesn't rely on some external structure)

# Alternatives

I've already described the two alternatives above: Action Functions vs
Action Objects.

# Unresolved questions

It might be desirable to make actions depend on other actions; e.g. a
multi step form that requires user interaction between N async steps
might benefit from a pattern whereby step2 action can't be performed 

I'd like to explore making actions dependent on the success/failure of
other objects, like dependent keys for actions; e.g. in a multi-step
process, it would be nice to prevent `step2` action from being
performable unless `step1` has fulfilled, and possibly expose additional
state such as `{{#if step2.available}}` while letting the
controller/component JS code describe the dependencies between the actions.

There is also the question of whether we want sugar to automatically
pass route functions to controllers/components, but that might be beyond
the scope of this RFC.



