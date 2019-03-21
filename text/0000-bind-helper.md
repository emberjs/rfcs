- Start Date: 2019-03-20
- Relevant Team(s): Ember.js, Learning
- RFC PR: https://github.com/emberjs/rfcs/pull/470
- Tracking: (leave this empty)

# Bind Helper

## Summary

This RFC introduces a new helper `bind` to allow clear argument and context scope binding for functions passed in templates.

## Motivation

The current `action` helper has a lot of implicit and confusing behavior that is different than the Octane and post Octane programming model.

To understand the complexity of `action` there are many complex behaviors including:

1. Argument partial application (currying)
2. `this` context binding
3. `send` checks for Component and Controllers

At build time the `action` helper currently is passed through an AST transform to explicitly bind the `this` to be deterministic at runtime. This is a private API where the outputted Glimmer is not a 1-1 to the template. Also, the `action` helper is confused and has overlap with the `action` modifier which has similar but slightly different behavior.

Instead of this confusing and overloaded behavior, a new `bind` helper would be introduced to do partial application and context binding (with no need for build time private APIs).

## Detailed design

The `bind` helper will take in a function and then the set of arguments that will be partially applied to the function.
As an optional named argument the user can pass in a `this` property that will set the `this` context this named argument would default to `undefined`.

The behavior of this helper will be simple and use JavaScript's `function.prototype.bind`.

Here are some examples of the `bind` helper and the equivalent JS:

### Simple Case On Argument Curry

```hbs
{{bind this.log 1}}
```

```js
this.log.bind(null, 1);
```

### Multiple Argument Partial Application

```hbs
{{bind this.add 1 2}}
```

```js
this.add.bind(null, 1, 2);
```

### Setting the `this` context

```hbs
{{{bind this.session.logout context=this.anotherSession}}}
```

```js
this.session.logout.bind(this.session)
```

### Setting the `this` context and arguments

```hbs
{{bind this.car.drive 1000 'mile' context=this.honda}}
```

```js
this.car.logout.bind(this.car, 1000, 'mile')
```

### Comparison to Action Helper/Modifier

```hbs
{{!-- Actions --}}
<button {{action "increment"}}>Click</button>
<button {{action this.increment}}>Click</button>
<button onclick={{action "increment"}}>Click</button>
<button onclick={{action this.increment}}>Click</button>
<button {{action (action "increment")}}>Click</button>
<button {{action (action this.increment)}}>Click</button>

<button onclick={{bind this.increment context=this}}>Click</button>
<button {{on "click" (bind this.increment context=this)}}>Click</button>
```

### With `mut`

```hbs
{{!-- Actions --}}
<button {{action (mut showModal) true}}>Click</button>
<button onclick={{action (mut showModal) true}}>Click</button>
<button {{action (action (mut showModal) true)}}>Click</button>

<button onclick={{bind (mut showModal) true}}>Click</button>
<button {{on "click" (bind (mut showModal) true)}}>Click</button>
```

Because `function.prototype.bind` always binds the `this` context the bind helper would require functions to be already bound or the `this` named argument would need to be explicitly set.

## How we teach this

For guides we would switch to recommending the `bind` helper to pass functions into components args and modifiers.
In guides we would no longer recommend using the `action` helper based on the reasons listed in motivations.

## Drawbacks

Since this recommended design sets the `this` context to `null` using `function.prototype.bind` this could be confusing to junior developers if the function was not already bound (using something like an `@bound` decorator, JS binding, arrow functions, or the `bind` helper earlier in the stack).

## Alternatives

One alternative would be to continue using the `action` helper despite confusion and overloading behavior.

Another alternative would be to make the `bind` helper arguments exactly match `function.prototype.bind` (requiring `this` to always be passed in as the first argument). Doing this should probably introduce an `unbound` helper to allow partial argument application without changing `this`.

A third alternative would be to make the `context=` behavior not bind the `this` context. The ability to do this would need some introspection of what the context is of the function call (essentially recreating the complexity of `action`).

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
How does this feel with `mut` since we do not `value=` syntax (possibly an `extract` helper or some syntax to get args from events or nested objects).
