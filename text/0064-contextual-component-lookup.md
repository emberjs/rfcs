---
Start Date: 2015-06-12
RFC PR: https://github.com/emberjs/rfcs/pull/64
Ember Issue: (leave this empty)

---

# Summary

The goal of this RFC is to allow for better component composition and the
usage of components for domain specific languages.

Ember components can be invoked three ways:

* `{{a-component`
* `{{component someBoundComponentName`
* `<a-component` (coming soon!)

In all these cases, attrs passed to the component must be set at the place of
invocation. Only the `{{component someBoundComponentName` syntax allows for the name
of the component invoked to be decided elsewhere.

All component names are resolved to components through one global resolution
path.

To improve composition, four changes are proposed:

* The `(component` helper will be introduced to close over component attrs in
  a yielding context.
* The `{{component` helper will accept an argument of the object created by
  `(component` for invocation (as it invokes strings today).
* Property lookups with a value containing a dot will be considered for
  rendering as components. `{{form.input}}` would be considered, for instance.
  Helper invocations with a dot will also be treated like a component if the
  key has a value of a component, for instance `{{form.input value=baz}}`.
* A `(hash` helper will be introduced.

# Motivation

When building a complex UI from several components, it can be difficult to
share data without breaking encapsulation. For example this template:

```hbs
{{#great-toolbar role=user.role}}
  {{great-button role=user.role}}
{{/great-toolbar}}
```

Causes the user to pass the `role` data twice for what are obviously related
components. A component can yield itself down:

```hbs
{{! app/components/great-toolbar/template.hbs }}
{{yield this}}
```

```hbs
{{#great-toolbar role=user.role as |toolbar|}}
  {{great-button toolbar=toolbar}}
{{/great-toolbar}}
```

And `great-button` can have knowledge about properties on `great-toolbar`, but
this break the isolation of components. Additionally the calling syntax is not
much better, `toolbar` must still be passed to each downstream component.

Often `nearestOfType` is used as a workaround for these limitations. This API
is poorly performing, and still results in the downstream child accessing the
parent component properties directly.

Consequently there is a demand by several addons for improvement. Our goal
is a syntax similar to DSLs in Ruby:

```hbs
{{#great-toolbar role=user.role as |toolbar|}}
  {{toolbar.button}}
  {{toolbar.button orWith=additionalProperties}}
{{/great-toolbar}}
```

As laid out in this proposal, the `great-toolbar` implementation would look
like:

```hbs
{{! app/components/great-toolbar/template.hbs }}
{{yield (hash
  button=(component 'great-button' role=user.role)
)}}
```

# Detailed design

### The `(component` helper and `{{component` helper

Much like `(action` creates a closure, it is proposed that the `(component`
helper create something similar. For example with actions:

```hbs
{{#with (action "save" model) as |save|}}
  <button {{action save}}>Save</button>
{{/with}}
```

The returned value of the `(action` nested helper (a function) closes over the
action being called (`actions.save` on the context and the `model` property).
The `{{action` helper can accept this resulting value and invoke the action
when the user clicks.

The `(component` helper will close over a component name. The
`{{component` helper will be modified to accept this resulting value and invoke
the component:

```hbs
{{#with (component "user-profile") as |uiPane|}}
  {{component uiPane}}
{{/with}}
```

Additionally, a bound value may be passed to the `(component` helper. For
example `(component someComponentName)`.

Attrs for the final component can also be closed over. Used with yield, this
allows for the creation of components that have attrs from other scopes. For
example:

```hbs
{{! app/components/user-profile.hbs }}
{{yield (component "user-profile" user=user.name age=user.age)}}
```

```hbs
{{#user-profile user=model as |profile|}}
  {{component profile}}
{{/user-profile}}
```

Of course attrs can also be passed at invocation. They smash any conflicting
attrs that were closed over. For example `{{component profile age=lyingUser.age}}`

Passing the resulting value from `(component` into JavaScript is permitted,
however that object has no public properties or methods. Its only use would
be to set it on state and reference it in template somewhere.

### Hash helper

Unlike values, components are likely to have specific names that are semantically
relevent. When yielded to a new scope, allowing the user to change the name
of the component's variable would quickly lead to confusing addon documentation.
For example:

```hbs
{{#with (component "user-profile") as |dropDatabaseUI|}}
  {{component dropDatabaseUI}}
{{/with}}
```

The simplest way to enforce specific names is to make building hashes
of components (or anything) easy. For example:

```hbs
{{#with (hash profile=(component "user-profile")) as |userComponents|}}
  {{component userComponents.profile}}
{{/with}}
```

The `(hash` helper is a generic builder of objects, given hash arguments. It
would also be useful in the same manner for actions:

```hbs
{{#with (hash save=(action "save" model)) as |userActions|}}
  <button {{action userActions.save}}>Save</button>
{{/with}}
```

### Component helper shorthand

To complete building a viable DSL, `.` invocation for `{{` components will be
introduced. For example this `{{component` invocation:

```hbs
{{#with (hash profile=(component "user-profile")) as |userComponents|}}
  {{component userComponents.profile}}
{{/with}}
```

Could be converted to drop the explicit `component` helper call.

```hbs
{{#with (hash profile=(component "user-profile")) as |userComponents|}}
  {{userComponents.profile}}
{{/with}}
```

A component can be invoked like this only when it was created by the
`(component` nested helper form. For example unlike with the `{{component`
helper, a string is not acceptable.

To be a valid invocation, one of two criteria must be met:

* The component can be called as a path. For example `{{form.input}}` or `{{this.input}}`
* The component can be called as a helper. For example `{{form.input value=baz}}` or `{{this.input value=baz}}`

And of course a `.` must be present in the path.

# Drawbacks

This proposal encourages aggressive use of the `(` nested helper syntax.
Encouraging this has been slightly controversial.

No solution for angle components is presented here. The syntax for `.`
notation in angle components is coupled to a decision on the syntax for
bound, dynamic angle component invocation (a `{{component` helper for angle
components basically).

`(component 'some-component'` may be too verbose. It may make sense to simply
allow `(some-component`.

Other proposals have leaned more heavy on extending factories in JavaScript
then passing an object created in that space. Some arguments against this:

* Getting the container correct is tricky. Who sets it when?
* Properties on the classes would not be naturally bound, as they are in this proposal.
* As soon as you start setting properties, you likely want a `mut` helper,
  `action` helper, etc, in JavaScript space.
* Keeping the component lookup in the template layer allows us to take advantage
  of changes to lookup semantics later, such as local lookup in the pods
  proposal.

# Alternatives

All pain, no gain. Addons really want this.

# Unresolved questions

There has been discussion of if a similar mechanism should be available for
helpers.
