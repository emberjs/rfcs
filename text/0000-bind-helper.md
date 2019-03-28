- Start Date: 2019-03-20
- Relevant Team(s): Ember.js, Learning
- RFC PR: https://github.com/emberjs/rfcs/pull/470
- Tracking: (leave this empty)

# `{{with-args}}` Helper

## Summary

This RFC introduces a new helper, `{{with-args}}`, to allow clear argument passing for functions in templates.

## Motivation

The current `action` helper has a lot of implicit and confusing behavior that is different than the Octane and post Octane programming model.

To understand the complexity of `action` there are many complex behaviors including:

1. Argument partial application (currying)
2. `this` context binding
3. `send` checks for Component and Controllers

At build time the `action` helper currently is passed through an AST transform to explicitly bind the `this` to be deterministic at runtime. This is a private API where the outputted Glimmer is not a 1-1 to the template. Also, the `action` helper is confused and has overlap with the `action` modifier which has similar but slightly different behavior.

Instead of this confusing and overloaded behavior, a new `with-args` helper would be introduced to do partial application (with no need for build time private APIs), and context binding will be done instead using the `@action` decorator in classes.

## Detailed design

The `with-args` helper will take in a function and then the set of arguments that will be partially applied to the function.

Here are some examples of the `with-args` helper and the equivalent JS:

### Simple Case On Argument Curry

```hbs
{{with-args this.log 1}}
```

```js
return function() {
  this.log.call(this, 1);
}
```

### Multiple Argument Partial Application

```hbs
{{with-args this.add 1 2}}
```

```js
return function() {
  this.log.call(this, 1, 2);
}
```

The use of `function` application like so allows us to preserve/pass through the `this` context of the calling site accurately, so creating a function `with-args` is equivalent to the same function _without_ args.

We will also reserve the `withArgs` helper name for future use as an alias, anticipating changes coming from template imports and most helpers/components conforming to valid JS identifiers.

### Comparison to Action Helper/Modifier

```hbs
{{!-- Actions --}}
<button {{action "increment" 5}}>Click</button>
<button {{action this.increment 5}}>Click</button>
<button onclick={{action "increment" 5}}>Click</button>
<button onclick={{action this.increment 5}}>Click</button>
<button {{action (action "increment" 5)}}>Click</button>
<button {{action (action this.increment 5)}}>Click</button>

<button onclick={{with-args this.increment 5}}>Click</button>
<button {{on "click" (with-args this.increment 5)}}>Click</button>
```

### With `mut`

```hbs
{{!-- Actions --}}
<button {{action (mut showModal) true}}>Click</button>
<button onclick={{action (mut showModal) true}}>Click</button>
<button {{action (action (mut showModal) true)}}>Click</button>

<button onclick={{with-args (mut showModal) true}}>Click</button>
<button {{on "click" (with-args (mut showModal) true)}}>Click</button>
```

## How we teach this

For guides we would switch to recommending the `with-args` helper to pass functions into components args and modifiers.

In guides we would no longer recommend using the `action` helper based on the reasons listed in motivations.

## Drawbacks

* `with-args` is a bit of a verbose name, and given this is a commonly used helper it could be a papercut to users and make templates harder to read. Combined with the extra complexity of requiring the `@action` decorator to bind context, and the `{{on}}` modifier to trigger events, it could lead to users avoiding the helper altogether in favor of `{{action}}`.

## Alternatives

One alternative would be to continue using the `action` helper despite confusion and overloading behavior.

There are also a number of potential alternatives for names:

* `args` - A shorter, simpler name with similar properties. This is somewhat less self-explanatory (on its own, without context, one might think it refers to component args, or does something with them), but may make up for this by being short and simple.
* `bind` - The original name this RFC suggested. `bind` is fairly _imperative_, it describes the action that we do to the function rather than what is returned. It also does not exactly match the JS method API, and as noted in the RFC feedback this could be confusing. Finally, it requires stopping to teach the concept of binding and how that works, which is a lot of overhead for a helper that will be used early and often.
* `call` - This reads nicely in templates, but is _very_ imperative and has already been confusing to folks when discussed. It differs significantly from the JS method API, teaching around this would be difficult, though possible. It also is not clear that it is _not required_ unless args are being passed, so we may see users attempting to use it for plain functions.

Names that have been considered, but passed over:

* `apply` - Same downsides as `call`, but less nice to read.
* `applyArgs` - Similar enough to `with-args`, but uses more obscure computer-science-y terminology without many benefits.
* `partial` - From LISP and other languages. `partial` there means "Partial Application". This is a computer-science-y term that isn't super explanatory, plus `partial` is already a (deprecated) feature in Ember templates.
* `papply` - From R. Generally unaesthetically pleasing, and same issues as `partial`
* `action` - We considered trying to reclaim the "action" term, but it still has the same problems of overlap with the modifier and decorator, and there isn't an easy transition path to deprecating the automatic `this` binding.
* `callback` - Considered, but it conflicts with the `@action` decorator's naming - the method is an action, until we pass it to callback, at which point it's a callback? This felt too confusing, and we believe it makes most sense if the helper is an "adjective" that modifies whatever its input is.
