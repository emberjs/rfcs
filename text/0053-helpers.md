---
Start Date: 2015-05-17
RFC PR: https://github.com/emberjs/rfcs/pull/53
Ember Issue: https://github.com/emberjs/ember.js/pull/11278

---

# Summary

Ember.js 1.13 will introduce a new API for helpers. Helpers will come in two
flavors:

**Helpers** are a class-based way to define HTMLBars subexpressions. Helpers:

  * Have a single return value.
  * Must have a dash in their name.
  * Cannot be used as a block (`{{#some-helper}}{{/some-helper}}`).
  * Can store and read state.
  * Have lifecycle hooks analogous to components where appropriate. For
    example, a helper may call `recompute` at any time to generate a new
    value (this is akin to `rerender`).
  * Are a superset of shorthand helpers, the function-based syntax described
    below. They can do more, but in many cases a shorthand helper is appropriate.

**Shorthand helpers** are a function-based way to define HTMLBars
subexpressions. Helpers written this way:

  * Have all the limitations of regular helpers.
  * Have no instance associated with them, cannot store or read state.
  * Have no lifecycle hooks. The function is simply re-computed when any input
    changes.

These improved helpers fill a gap in Ember's current template APIs:

|                   | has positional params | has layout (shadow DOM) | can yield template | has lifecycle, instance | can control rerender |
|-------------------|-----------------------|-------------------------|--------------------|-------------------------|----------------------|
| components        | Yes                   | Yes                     | Yes                | Yes                     | Yes                  |
| helpers           | Yes                   | No                      | No                 | Yes                     | Yes                  |
| shorthand helpers | Yes                   | No                      | No                 | No                      | No                   |

An example helper:

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

An example shorthand helper:

```js
// app/helpers/full-name.js
import Ember from "ember";

export default Ember.Helper.helper(function(params, hash) {
  let fullName = params.join(' ');
  if (hash.honorific) {
    fullName = `${hash.honorific} ${fullName}`
  }
  return fullName;
});
```

Helpers can be used anywhere an HTMLBars subexpression is valid:

```hbs
{{full-name 'Bigtime' 'Beagle'}}
{{input value=(full-name 'Gyro' 'Gearloose') readonly=true}}
{{#if (eq (full-name 'Webbigail' 'Vanderquack') selectedFullName))}}
  You have chosen wisely.
{{/if}}
```

# Motivation

Ember.js 1.13 make a private API change that removed the ability to access
application containers. `Ember.HTMLBars._registerHelper` was previously passed
the `env` object, and this was removed as it is an internal implementation
detail.

Ember's helper API has not kept pace with improvements possible
after the introduction of HTMLBars. This has resulted in the community using
a variety of private APIs, many of which leak information about the outer
context of a helpers invocation as well as the render layer implementation.

The current public API is:

* [Ember.Handlebars.makeBoundHelper](http://emberjs.com/api/classes/Ember.Handlebars.html#method_makeBoundHelper)

This API is sorely lacking in functionality required by addon authors.

* Has no access to other parts of the app, like services
* Leaks a private API for dealing with blocks
* Results in less efficient helpers due to the Handlebars compatibility layer
* Has poor support for hash arguments

Additionally it remains difficult to write a helper that recomputes due to
something besides the change of its input.

Specifically, this RFC addresses many of the concerns in
[emberjs/ember.js#11080](https://github.com/emberjs/ember.js/issues/11080).
Libraries such as [yahoo/ember-intl](https://github.com/yahoo/ember-intl),
[dockyard/ember-cli-i18n](https://github.com/dockyard/ember-cli-i18n), and
[minutebase/ember-can](https://github.com/minutebase/ember-can) will be
provided a viable public API to couple to.

# Detailed design

Helpers must have a dash in their name. In an Ember-CLI app, they can be named
according to the `app/helpers/full-name.js` convention (`app/full-name/helper.js`
in pods mode). For a globals app, naming a helper `App.FullNameHelper` is
sufficient.

### Definition and lifecycle

A helper is defined as a class inheriting from `Ember.Helper`. For
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
  value is set to the class.

The `compute` function is always called with `params` (the bare, ordered
arguments) and `hash` (the named arguments). For example:

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

Which functions the same as this shorthand:

```js
// app/helpers/greet-someone.js
import Ember from "ember";

// Usage: {{greet-someone 'bob' greeting='say hello'}}
export default Ember.Helper.helper(function(params, hash) {
  return `Hello ${params[0]}, nice to ${hash.greeting}`;
});
```

When the `params` or `hash` contents change, the `compute` method is called
again. The instance of the helper is preserved across rerenders of the parent.
A shorthand helper, having no instance, is called every time a bound
argument changes.

The `init` and `destroy` methods can be subclassed for setup and teardown.

### Consuming a helper

Helpers can be used anywhere an HTMLBars subexpression can be
used. For example:

```hbs
{{#if (can-access 'admin')}}
  {{link-to 'login'}}
{{/if}}
{{#if (eq (can-access 'admin') false)}}
  No login for you
{{/if}}
<my-login-button isAdmin={{can-access 'admin'}} />
Can access? {{can-access 'admin'}}
```

Passing a helper to a `{{`- invoked component skips the auto-`mut` behavior:

```hbs
{{my-login-button isAdmin=(can-access 'admin')}}
```

Let's step through exactly what happens when using an helper like this:

```hbs
<my-login-button isAdmin={{can-access 'admin'}} />
```

Upon initial render:

* The helper `can-access` is looked up on the container
* The helper is identified as a full helper, not a shorthand helper function
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

The return value of helper is passed through to where their subexpression
is called. For example, given a helper (this one a shorthand helper):

```js
// app/helpers/full-name.js
import Ember from "ember";

export default Ember.Helper.helper(function fullName(params, hash) {
  return params.join(' ');
}
```

The following are effectively the same:

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

This is a legacy form of mustache usage. Helpers will throw an exception when
used in this manner.

### Consuming services and recompute

Helpers are a valid target for service injection. For example:

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

For this reason, helpers are granted some control over their
computation lifecycle. A helper will recompute when:

* A value passed via the template changes (`params` or `hash`)
* The `recompute` method is called

For example, this helper checks if the current use has access to a
resource type:

```js
// app/helpers/can-access.js
import Ember from "ember";

// Usage {{if (can-access 'admin') 'Welcome, boss' 'Heck no!'}}
export default Ember.Helper.extend({
  session: Ember.inject.service(),
  onCurrentUserChange: Ember.observes('session.currentUser', function() {
    this.recompute();
  }),
  compute(params) {
    const currentUser = this.get('session.currentUser');
    return currentUser.can(params[0]);
  }
});
```

# Drawbacks

Helpers may superficially appear similar to components, but in
practice they have none of the special behavior of components such as managing
DOM. The intent of this RFC is that full class-based helpers remain very close
to the spirit of a pure function (as in the shorthand). However, despite this
intent they are a new concept for the framework.

# Alternatives

A [previous RFC](https://github.com/emberjs/rfcs/pull/52) explored creating a new class called Expressions, which would have
more closely modeled the API of components (using positional params, attrs).
After discussion and consideration it was clear that a third kind of template
API would be very challenging to document and teach well.

# Unresolved questions

Perhaps there should be hooks in place for the lifecycle, instead of relying on
`init` and `destroy`.
