# RFC for Hook/Event for Transitioning During a Transition

- Start Date: 2016-03-14
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

The transition lifecycle currently has `willTransition` and `didTransition` hooks that we can use to respond to or modify behavior of route transitions. We also encourage modifying these transitions while they're in flight using things like `transition.abort` and `Route#transitionTo` inside of `Route#redirect`, amongst other places.

We need the ability to be notified when any of these eventualities occur so that we can respond to those as well.

# Motivation

Being able to track the entire transition lifecycle allows us the ability to provide much richer interactions:

- Tracking is one of the first things that many people implement inside of Ember using `didTransition`. That simplistic approach covers many scenarios, but more advanced tracking will want to be able to attach information from what initiated the transition to the final consequence of the navigation. Multiple redirects along the way will be lost (A, click to load B, B redirects to C, C redirects to D, D redirects to E, we only capture first and last edge) and the simplistic approach of "attaching rider information to the `transition` object" won't survive any redirections or be present during `intermediateTransitionTo`.
- Animations can be handled in a complex way using `liquid-fire` but not without non-trivial performance impact for setting up new HTMLBars block contexts. The complex mechanisms rely on information gleaned from the rendering lifecycle, but a simplistic version could be implemented solely using information from the transition lifecycle if given the ability to detect modifications to the transition.
- Knowing how and in what way to handle parallelizing data fetching requires pre-knowledge of the destination route, and updating knowledge about the destination route along the way. We can optimistically load data for the first target, and evict from the request queue if the `transition` changes.
- Accessibility via `ember-a11y` needs knowledge of changes in transitions in order to identify when it needs to reset state to its previous container or set focus to an element responsible for loading the route.

# Detailed design

The `willChangeTransition` event should be fired for every new transition object created between firing `willTransition` and `didTransition`.

```javascript
const Router = Ember.Router.extend({
  onWillTransition: Ember.on('willTransition', function(transition) {
    console.log(`transitioning to "${transition.targetName}"`);
  }),

  onWillChangeTransition: Ember.on('willChangeTransition', function (transition) {
    console.log(`redirecting to "${transition.targetName}"`);
  }),

  onDidTransition: Ember.on('didTransition', function() {
    console.log('transition settled');
  }),
});

Router.map(function() {
  this.route('first');
  this.route('second');
});

App.FirstRoute = Ember.Route.extend({
  redirect() {
    this.replaceWith('second');
  },
});

App.SecondRoute = Ember.Route.extend();

visit('/first');
// => transitioning to "first"
// => redirecting to "second"
// => transition settled
```

`willChangeTransition` should also be available as an action on routes. For a given sequence of transitions, the `willChangeTransition` action is triggered on the origin route, the same route on which `willTransition` is triggered.

```javascript
App.ApplicationRoute = Ember.Route.extend({
  actions: {
    willChangeTransition(transition) {
      // do stuff...
    },
  },
})
```

The `willChangeTransition` hook should provide the same API surface as the `willTransition` hook, and the callback for the `willChangeTransition` event/action should be passed the same parameters as the callback for the `willTransition` event/action. This grants users wishing to subscribe to all transition events the ability to reuse one function across both hooks or both callbacks.

```javascript
function hookForAllTransitions(oldInfos, newInfos, transition) {
  const ret = this.super(...arguments);

  // do stuff for all transitions...

  return ret;
}

function callbackForAllTransitions(transition) {
  // do stuff for all transitions...
}

const Router = Ember.Router.extend({
  willTransition: hookForAllTransitions,
  willChangeTransition: hookForAllTransitions,

  onWillTransition: Ember.on('willTransition', callbackForAllTransitions),
  onWillChangeTransition: Ember.on('willChangeTransition', callbackForAllTransitions),
});
```

This will require changes to both Ember and tildeio/router.js.

# How We Teach This

> What names and terminology work best for these concepts and why?

We propose an event name of `willChangeTransition` as parallel to the existing `willTransition` and `didTransition` hooks. It is understandable in context, and out of context while hewing close to existing patterns.

> How is this idea best presented? As a continuation of existing Ember patterns, or as a wholly new one?

This is new functionality but it was merely omitted from event handling in the existing implementation. There are already logical places inside of Ember to insert it, and the existing Ember patterns apply.

> Would the acceptance of this proposal mean the Ember guides must be re-organized or altered? Does it change how Ember is taught to new users at any level?

The guides would not need to be altered as this is a concept better suited for API-level documentation. New users will be unlikely to encounter this, and addon developers are likely the ones who most desire this feature.

> How should this feature be introduced and taught to existing Ember users?

The people who need this hook will seek it out. The target audience for this is implementors of addons as it generally will be used for changes across the entire application and not in one individual scenario. As such simply mentioning it in the Ember.js release blog entry and documenting it in the API will be sufficient.

# Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember, on the integration of this feature with other existing and planned features, on the impact of the API churn on existing apps, etc.

As this is a pure addition there will be no impacts to existing applications. There is a small increase in the public API of Ember, and a few more lines of code which we have to ship to application consumers. This is a relatively mature area of Ember that hasn't undergone significant change and as such it's unlikely to conflict with future plans.

> There are tradeoffs to choosing any path, please attempt to identify them here.

This is a minimally invasive change and we don't see anything particularly negative about adding this API.

# Alternatives

> What other designs have been considered? What is the impact of not doing this?

Another way to achieve this goal is to monkeypatch `Ember.Router`, but doing so is fragile. It results in unnecessary churn inside of the addon and undue effort for addon consumers. This is already implemented inside of the `ember-prefetch` addon and is about to be ported to `ember-a11y` in order to accomplish the same set of goals.

# Unresolved questions

This is a full design with no unresolved questions.
