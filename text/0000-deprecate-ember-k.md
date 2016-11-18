- Start Date: 2016-11-18
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

The `Ember.K` utility function is a low level utility that has lost most of its value today,
like the wisdom tooth of the frameworks were we only think of it when it gives us problems.

# Motivation

Let's start explaining what is `Ember.K`.

It is this function:

```js
Ember.K = function() {
  return this;
}
```

The purpose of this utility was to avoid some amount of boilerplate code
and limit the creation of function instances in Ember's internals.

In a world of globals, effectively writing `somefn: Ember.K` was shorter
than writing

```js
someFn: function() {
  return this;
}
```
and generated less function alocations.

However with the introduction of ES6 modules and the modularization of Ember
in process (#176), keeping this feature would require to design an import path for it.

While doable, any possible benefit in code size or allocations obtained by reusing
the exact same Function instance everywhere would be greatly smashed by the
overhead both in space and CPU cycles of importing it in AMD-transpiled modules.
It's worth noting that ES6 shorthand method syntax has made writing functions
less verbose and those functions gain an internal name property that shows in
stack traces making debugging easier.

The second downside of reusing the same instance in many places is that if for
some reason the VM deoptimizes that function, that deoptimization is spreaded
across all the usages of `Ember.K`.

Lastly, the chainable nature of `Ember.K` tends to surprise the users:

```js
let derp = {
  foo: Ember.K,
  bar: Ember.K,
  baz: Ember.K
}

derp.foo().bar().baz(); // O_o
```

# Transition Path

The obvious first step is to make sure Ember, Ember-data and other pieces of the
ecosystem don't use `Ember.K.` internally.

The suggested transition is to *intentionally* not give `Ember.K` an import path in the new JS modules
being discussed in #176 or give it a import path that is clearly private.

Part of that RFC states that some sort of shim mode will allow people to keep using
`import Ember from 'ember';` for a while. Is in that shim where we could put a deprecation:

```js
Ember = {};
Ember.computed = ...
// more shims ...
Ember.K = deprecatedVersionOfEmberK.
```

As people migrate to the ES6 modules they will have to update their code since `Ember.K`
cannot be used without using the shim.

Perhaps the codemod that will perform this transition should be aware of this case so usages of `Ember.K`
are replaced by the import from a private path or maybe go one step beyond and replace
it by an empty function.

# How We Teach This

Since it is a very low-level utility the amount of people that will have to
update their code to stop using it is rather small, and probably only include
seasoned Ember developers, so with deprecating the feature in the guides.

If this is done, as suggested, as part of #176, it will be mentioned in the
document or blog post announcing the final transition to modules.


# Drawbacks

Althoug this utility is not very used, there is a chance that is used by some
addons and as a placeholder of a hook that is called a lot and would trigger
hundreds of deprecation warnings.

# Alternatives

The feature could continue to exist.

