- Start Date: 2015-10-27
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Provide access to `hasBlock` in component's javascript definition.

# Motivation

I believe there are some valid use cases for this feature. Currently it is available in component's templates, but not in the javascript definition of them. In a component template `hasBlock` is true when the component was invoked with a block. Refer to [`hasBlock` API docs](http://emberjs.com/api/classes/Ember.Component.html#property_hasBlock). Also, there is a closed issue on this: https://github.com/emberjs/ember.js/issues/11741.

My use case is basically a component that, when invoked in a block form, should run some logic (involving a 3rd party library) to initialize a popup.

Another probably more common use case would be to apply a css class or attribute in the component element based on if the component was invoked with a block or not.

I believe that this small change would enable richer components that could behave differently depending on how they're invoked.

# Detailed design

Currently, this value is "hidden" behind a `Symbol` (or at least our implementation of it), thus making it private.
The implementation would essentially consist in making it public, behind a feature flag.

# Drawbacks

I'm not aware of any drawbacks of this feature. Are we somehow promoting bad component design in any way?

# Alternatives

There is a very evil but smart alternative, suggested by @rwjblue, which consists in calling a computed property in a template, depending on `hasBlock`:

```hbs
{{#if hasBlock}}{{setHasBlock}}{{/if}}`
```

```js
setHasBlock: Ember.computed(function() {
  this.set('hasBlock', true);
})
```

This is of course not a decent final way to solve the problem, and it may not work in certain edge cases?
