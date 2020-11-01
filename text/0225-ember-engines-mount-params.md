---
Start Date: 2017-04-26
RFC PR: #15174
Ember Issue: (leave this empty)

---

# Summary

This RFC proposes allowing parameters to be passed to the `{{mount}}` syntax.

# Motivation

This will enable developers to pass contextual data into routeless engines at
runtime, allowing individual engines to be used multiple times through a single
application under different contexts.

An example could be a dashboard of charts where each chart is a routeless engine.
Each chart could be of a different type and would require different data. This
RFC would enable the following:

```hbs
{{!-- app/templates/application.hbs --}}
{{#each charts as |chart|}}
  {{mount "chart" type=chart.type data=chart.data}}
{{/each}}
```

# Detailed design

You can see the implementation for this RFC [here](https://github.com/emberjs/ember.js/pull/15174).

Implementing this functionality turns out to be relatively straight forward. With
routeless engines already supporting an application controller, we can use this
as a means of providing access to the parameters.

Parameters would be passed to the `{{mount}}` syntax in the same way that
they are currently passed to components and helpers.

```hbs
{{mount "foo" bar="baz"}}
```

These parameters would then be set as the `model` property on the engines
application controller; making them accessible from both a JS and HBS context.

```js
// foo/controllers/application.js
import Ember from 'ember';

export default Ember.Controller.extend({

  actions: {
    exampleAction() {
      let barParam = this.get("model.bar");
    }
  }

});
```

```hbs
{{!-- foo/templates/application.hbs --}}
The value of the bar param is: {{model.bar}}
```

# How We Teach This

This RFC re-uses concepts that are already heavily used throughout other areas
of the framework.

Updates will need to be made to the Ember API docs and [ember-engines.com](http://ember-engines.com) guides in order to explain that
routeless engines can now accept parameters being passed via the `{{mount}}` syntax.

In addition, updates would need to include examples of both how to pass parameters
to `{{mount}}` as well as how any passed parameters can be accessed from within
the context of an engine.

# Drawbacks

### Increased risk of runtime dependencies [[reference](https://github.com/ember-engines/ember-engines/issues/98#issuecomment-217347193)]
This RFC does increase the risk of introducing runtime dependencies.

Example:

```js
import Ember from 'ember';

export default Ember.Component.extend({
  geo: Ember.inject.service('geolocation')
})
```
```hbs
{{mount "blog" (hash geo=geo)}}
```

# Alternatives

### Application route `model` hook
This solution suggested that the parameters were provided to the engines
application route via the `model` hooks `params` argument. While viable,
this solution didn't _feel_ quite right as you were using a route within a _routeless_
engine.

### Introduction of new `routelessEngine` primitive
This solution suggested that routeless engines should be given their own primitive
similar to that of a component. The primitive would follow a construct to
components and use the same hooks `(e.g., didReceiveAttrs)` for working with
parameters.

This solution was decided against for following main reasons:

1. The current public API is that we use `application` controller to back the
`application.hbs` of a route-less engine.  Adding a new conceptual primitive
here would be a fairly difficult change without breakage.
2. Introducing a new primitive (e.g. not controller and not a component) is a
*very* heavy handed thing, and should not be taken lightly. We don't think this
rises to that level of need.
3. This is an ideal case for "just using a component", but that would be roughly
akin to "routable components" and we should follow Ember's lead.  When routeable
components are introduced, we can refactor this in the same way with the same
backwards compatibility guarantees.

# Unresolved questions

### Positional parameters
In addition to supporting named parameters, should the `{{mount}}` syntax also
support positional parameters? If so, should this be covered in this RFC, or
in a follow up RFC?
