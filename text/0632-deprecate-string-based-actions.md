- Start Date: 2020-05-23
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/632
- Tracking: (leave this empty)

# Deprecate String Based Actions

## Motivation

In Ember 1.13 we introduced [closure actions](https://github.com/emberjs/rfcs/blob/00ac2685c86f27d41547012903f485a4ef338d27/active/0000-improved-actions.md), which have been recommended over string based actions. This officially deprecates string based actions in favor of closure actions,
which provide many benefits such as

1. simpler passing through components
2. less need for an actions hash
3. ability to return a value
4. typing in TypeScript
5. more in line with the Octane programming model

## Detailed Design

We will deprecate the following:

1. The `send` api.
2. The use of a string parameter to the `action` helper.
3. The use of a string parameter to the `action` modifier.

The `send` api may currently be used in components, controllers, and routes.
It is used to target the `actions` hash. In calling the `actions` hash of the
same object, `send` can be replaced by a function call once the method is
moved outside the `actions` hash.

For cases where `send` is used to go from a controller to a route or from
a route to a parent route, no direct replacement for this "action bubbling"
behavior will be provided.
Instead, users are recommended to inject the router service.

## Transition Path

Users will transition to closure actions.

While the action helper and modifiers are still supported with non-string arguments, the refactoring recommendation is to switch to the on modifier and @action decorator directly. It is possible that the action helpers and modifiers will be deprecated in future RFCs.

For example, take this button that sends the foo action with the default click event:

```js
export default Component.extend({
    someMethod() {
        this.send('actionName');
    },

    actions: {
        actionName() {
            // do something
        }
    }
});
```

```hbs
<button {{action 'actionName'}}>Action Label</button>
```

Although this could be refactored to:


```js
export default Component.extend({
    someMethod() {
        this.actionName();
    },

    actionName() {
        // do something
    }
});
```

```hbs
<button {{action this.actionName}}>Action Label</button>
```

Because the action modifier might be deprecated in the future,
it is recommended that the code is refactored instead to:

```js
export default Component.extend({
    someMethod() {
        this.actionName();
    },

    actionName: action(function() {
        // do something
    })
});
```

```hbs
<button {{on "click" this.actionName}}>Action Label</button>
```

so that it can eventually become, once refactored to native classes:

```js
export default class extends Component {
    someMethod() {
        this.actionName();
    }

    @action
    actionName() {
        // do something
    }
}
```

```hbs
<button {{on "click" this.actionName}}>Action Label</button>
```

For the send api, it is now generally possible to avoid string based actions in the actions
hash of any route, because the router service has all the methods that were once only
available in routes. If there are any route actions, it is recommended to move them to a
component, service, or a controller.

For example, if you currently have a route action accessed by the `send` api like this:

```js
// app/routes/some-route.js

actions: {
    myAction() {
        this.transitionTo('some.other.route');
    }
}
```

and are calling it from using `send` in your controller like this:

```js
// app/controllers/some-route.js
someMethod() {
    this.send('myAction');
}
```

You can move the action from the route to the controller:

```js
// app/controllers/some-route.js
export default Controller.extend({
    router: service(),

    someMethod() {
        this.myAction();
    }

    myAction: action(function() {
        this.router.transitionTo('some.other.route');
    })
});
```

In native class syntax, this looks like:

```js
// app/controllers/some-route.js
export default class SomeRouteController extends Controller {
    @service router;

    someMethod() {
        this.myAction();
    }

    @action
    myAction() {
        this.router.transitionTo('some.other.route');
    }
}
```

If you are using `ember-route-action-helper`, you can instead
move your action from the route to the controller.

From:

```js
// app/routes/some-route.js
export default Route.extend({
    actions: {
        someRouteAction() {
            // do something
        }
    }
}
```

```hbs
<!-- app/templates/some-route.hbs -->
<SomeComponent myAction={{route-action 'someRouteAction'}} />
```

To:

```js
// app/routes/some-controller.js
export default class SomeController extends Controller {
    @action
    someAction() {
        // so something
    }
}
```

```hbs
<!-- app/templates/some-route.hbs -->
<SomeComponent myAction={{someAction}} />
```

## How We Teach This

We already teach closure actions exclusively in the guides. Other than a new
deprecation guide, no additional material should be necessary.

## Drawbacks

Some older applications have thousands of string based actions throughout their codebases.
We should consider writing a codemod to ease the transition.

## Alternatives

Closure actions are now the standard in Ember, so other alternatives have not been
seriously considered.

## Unresolved questions

Can we also deprecate the use of the `actions` hash? Can we also deprecate using `this.actions`?
Can the `target` property of `@ember/controller` and `@ember/component` be deprecated?
