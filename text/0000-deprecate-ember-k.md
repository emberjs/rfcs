- Start Date: 2016-11-18
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

The `Ember.K` utility function is a low level utility that has lost most of its value today.

# Motivation

Let's start explaining what `Ember.K` is.

It is an utility function to avoid boilerplace code and limit the creation of function instances
in Ember's internals. The source code for this API is the following:

```js
Ember.K = function() {
  return this;
}
```

In a world of globals, writing `somefn: Ember.K` was effectively shorter
than writing

```js
someFn: function() {
  return this;
}
```

and generated fewer function allocations.

However with the introduction of ES6 modules and the modularization of Ember
in process (#176), keeping this feature would require to design an import path for it.

While doable, the transpiled output is actually bigger then defining the functions
inline, specially with the ES6 shorthand method syntax, and the perf difference
of saving a few function allocations is despicable.

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

The necessary first step is to make sure Ember, Ember Data and other pieces of the
ecosystem don't use `Ember.K` internally.

Phased approach:
* Deprecate `Ember.K`: Use the deprecation API to signal the deprecation, and deprecation guide entry.
* Add rule to ember-watson
* Extract to addon. Precedence: `active-model-adapter`, `store.filter`
* Do not include in #176.

# How We Teach This

Since it is a very low-level utility,
the amount of people that will have to update their code should be a limited set of developers, working mostly on addons.
This allows us to cover most use cases with the following strategy:
* Improve the current documentation to help developers finding the API for the first time in the future;
* Introduce the mandatory entry in the deprecations guide
* Provide an automated path forward through tooling such as [ember-watson](https://github.com/abuiles/ember-watson).

If this RFC is done as part of #176 as suggested,
it will be in the document or blog post announcing the final transition to modules.

# Drawbacks

Although this utility is not very used, there is a chance that is used by some
addons and as a placeholder of a hook that is called a lot and would trigger
hundreds of deprecation warnings.

# Alternatives

The feature could continue to exist.

