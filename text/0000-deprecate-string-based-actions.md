- Start Date: 2020-05-23
- Relevant Team(s): Ember.js
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Deprecate String Based Actions

## Motivation

> In Ember 1.13 we introduced [closure actions](https://github.com/emberjs/rfcs/blob/00ac2685c86f27d41547012903f485a4ef338d27/active/0000-improved-actions.md) have been recommended over string based actions. This officially deprecates string based actions in favor of closure actions,
which provide many benefits such as

1. simpler passing through components
2. less need for an actions hash
3. ability to return a value
4. typing in TypeScript
5. compatible with Glimmer Components

## Detailed Design

We will deprecate the following:

1. The `send` api.
2. The use of a string parameter to the `action` helper.
3. The use of a string parameter to the `action` modifier.

## Transition Path

Users will transition to closure actions.

From:

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

To:

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

From:

```hbs
<button {{action 'actionName'}}>Action Label</button>
```

To:

```hbs
<button {{action this.actionName}}>Action Label</button>
```

## How We Teach This

> We already teach closure actions exclusively in the guides. Other than a new
deprecation guide, no additional material should be necessary.

## Drawbacks

> Some older applications have thousands of string based actions throughout their codebases.
We should consider writing a codemod to ease the transition.

## Alternatives

> Closure actions are now the standard in Ember, so other alternatives have not been
seriously considered.

## Unresolved questions

> Can we also deprecate the use of the `actions` hash? Can we also deprecate using `this.actions`?
