- Start Date: 2015-05-17
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Stateful helpers are a class-based way to define helpers. Stateful helpers:

  * Have a single return value
  * Can store and read state
  * Have lifecycle hooks analagous to components where appropriate. For
    example, a stateful helper may call `recompute` at any time to generate a new
    value.
  * Are a superset of traditional function-based helpers. They can do more,
    but in many cases a traditional helper is appropriate.

They fill a gap in Ember's current template APIs:

 |has positional params|has layout|can yield template|has lifecycle, instance|can control rerender
---|---|---|---|---
components|Yes|Yes|Yes|Yes|Yes
stateful helpers|Yes|No|Yes|Yes|Yes
function helpers|Yes|No|Yes|No|No

Example usage:

```js
// app/helpers/full-name.js
import Ember from "ember";

export default Ember.Helper.extend({
  nameBuilder: Ember.inject.service(),
  compute(params) {
    const builder = this.get('nameBuilder');
    return builder.fullName(params[0], params[1]);
  }
});
```

Use an stateful helper anywhere a traditional helper is valid:

```hbs
{{full-name 'Bigtime' 'Beagle'}}
{{input value=(full-name 'Gyro' 'Gearloose') readonly=true}}
```

# Motivation

Helpers in Ember are pure functions. This make them easy to reason about, but
also overly simplistic. It is difficult to write a helper that recomputes due to
something besides the change of its input, and helpers have no API for
accessing parts of Ember like services.

Specifically, this addresses many of the concerns in
[emberjs/ember.js#11080](https://github.com/emberjs/ember.js/issues/11080).
Libraries such as [yahoo/ember-intl](https://github.com/yahoo/ember-intl),
[dockyard/ember-cli-i18n](https://github.com/dockyard/ember-cli-i18n), and
[minutebase/ember-can](https://github.com/minutebase/ember-can) will be
provided a viable public API to couple to.

# Detailed design

Stateful helpers have the same naming requirements, usage, and file placement
as helpers today. They have similar lifecycle hooks to components.

### Naming and file placement

The requirement for naming a stateful helper are the same as those for existing
helpers:

* Helper names should use a `-`.
* Helper names that do not use a `-` must be registered via
  `Ember.Helpers.registerHelper`.
* In Ember-CLI, helpers live in `app/helpers/full-name.js`, or with pods in
  `app/full-name/helper.js`.

### Defininition and lifecycle

A stateful helper is defined as a class inheriting from `Ember.Helper`. For
example:

```js
// app/helpers/hello-world.js
import Ember from "ember";

// Usage: {{hello-world}}
export default Ember.Helper.extend({
  compute() {
    return "Hello Helper World";
  }
});
```

Upon initial render:

* The helper instance is created.
* The `compute` method is called. The return value is outputted where the
  helper is used. For example in `<div class={{some-helper}}></div>` the return
  value is set to the class. This is a slight change from current function
  helper behavior, but the two behaviors should be aligned. See "Returning
  a value" and "Block form" below.

The `compute` function is always called with the same arguments a function
helper is called with:

```js
// app/helpers/greet-someone.js
import Ember from "ember";

// Usage: {{greet-someone 'bob' greeting='say hello'}}
export default Ember.Helper.extend({
  compute(params, hash) {
    return `Hello ${params[0]}, nice to ${hash.greeting}`;
  }
});
```

When the `params` or `hash` contents change, the `compute` method is called
again. The instance of the helper is preserved across rerenders of the parent.

The `init` and `destroy` methods can be subclassed for setup and teardown.

`compute` is called with the same arguments as a function helper.

### Consuming an stateful helper

Stateful helpers can be used anywhere an HTMLBars subexpression can be
used. For example:

```hbs
{{#if (can-access 'admin')}}
  {{link-to 'login'}}
{{/if}}
{{my-login-button isAdmin=(can-access 'admin')}}
<my-login-button isAdmin={{can-access 'admin'}} />
Can access? {{can-access 'admin'}}
```

In all these cases, the helper is considered one-way. That is, it is a
readable value and not two-way bound (toggling `isAdmin` does not impact the
helper).

Let's step through exactly what happens when using an helper like this:

```hbs
<my-login-button isAdmin={{can-access 'admin'}} />
```

Upon initial render:

* The helper `can-access` is looked up on the container
* The helper is identified as a stateful helper, not a function helper
* The helper is initialized (`init` is called)
* The `compute` function is called on the helper.
* The return value from `compute` is passed as an `attr` to `my-login-button`.
* The helper instance remains in memory.

If the parent scope is rerendered:

* The `compute` function is called again.
* The return value from `compute` is passed as an `attr` to `my-login-button`.

Upon teardown:

* The helper is destroyed, calling the `destroy` method.

### Returning a value

The return value of stateful and function helpers should be the return value
of the HTMLBars subexpression they are powering. For example, given a helper:

```js
// app/helpers/full-name.js
import Ember from "ember";

export default function full-name(params, hash, options) {
  return params.join(' ');
}
```

Or its equivilent stateful version, the following are effectively the same:

```hbs
<div data-name={{full-name "Fenton" "Crackshell"}}></div>
<div data-name={{"Fenton Crackshell"}}></div>
```

```hbs
{{my-component name=(full-name "Magica" "De Spell")}}
{{my-component name="Magica De Spell"}}
```

```hbs
<p>{{full-name "Bentina" "Beakley"}}</p>
<p>{{"Bentina Beakley"}}</p>
```

An exclusion to this pattern is the following form:

```hbs
<div {{full-name "Webbigail" "Vanderquack"}}></div>
```

This is a legacy form of mustache usage. In 2.0 both forms of helpers should
throw an exception when used in this manner.

### Block form

Helpers can optionally take a template and inverse block. To render a passed
block, they must yield it. For example, an `if-is-yes` helper might be
written like so:

```js
// app/helpers/custom-if.js
import Ember from "ember";

// Usage: {{if-is-yes 'no'}}not rendered{{else}}rendered!{{/if-is-yes}}
export default function ifIsYes(params, hash, options) {
  if (params[0] === 'yes') {
    options.template.yield();
  } else {
    options.inverse.yield();
  }
};
```

Porting this usage to a stateful helper only changes the boilerplate:

```
// app/helpers/custom-if.js
import Ember from "ember";

// Usage: {{if-is-yes 'no'}}not rendered{{else}}rendered!{{/if-is-yes}}
export default Ember.Helper.extend({
  compute(params, hash, options) {
    if (params[0]) {
      options.template.yield();
    } else {
      options.inverse.yield();
    }
  }
});
```

### Consuming services and recompute

Stateful helpers are a valid target for service injection. For example:

```js
// app/helpers/current-user-name.js
import Ember from "ember";

export default Ember.Helper.extend({
  // Same API as components:
  session: Ember.inject.service(),
  compute() {
    return this.get('session.currentUser.name');
  }
});
```

However consuming a property from a service does not bind the data being
displayed to that property. After `{{current-user-name}}` has been computed
and rendered, it will never be invalidated.

For this reason, stateful helpers are granted some control over their
computation lifecycle. A stateful helper will recompute when:

* A value passed via the template changes (`params` or `hash`)
* The `recompute` method is called
* Any key in the `recomputeObserves` array changes

For example, this stateful helper checks if the current use has access to a
resource type:

```js
// app/helpers/can-access.js
import Ember from "ember";

// Usage {{if (can-access 'admin') 'Welcome, boss' 'Heck no!'}}
export default Ember.Helper.extend({
  session: Ember.inject.service(),
  recomputeObserves: ['session.currentUser'],
  compute(params) {
    const currentUser = this.get('session.currentUser');
    return currentUser.can(params[0]);
  }
});
```

`recomputeObserves` can be desugared to:

```js
  recomputeWhen: Ember.observes('session.currentUser', function() {
    this.recompute();
  }),
```

# Drawbacks

Stateful helpers may superficially appear similar to components, but in
practice they have none of the special behavior of components. The intent of
this RFC is that they remain conceptually very close to function helpers,
but despite this intent they are a new concept for the framework.

# Alternatives

A [previous RFC](https://github.com/emberjs/rfcs/pull/52) explored creating a new class called Expressions, which would have
more closely modeled the API of components (using positional params, attrs).
After discussion and consideration it was clear that a third kind of template
API would be very challenging to document and teach well.

# Unresolved questions

Helpers are one-way. Is it possible to have a two-way helper (data
flowing upward like a mut)?

Perhaps there should be hooks in place for the lifecycle, instead of relying on
`init` and `destroy`.
