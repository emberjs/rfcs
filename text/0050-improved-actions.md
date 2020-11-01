---
Start Date: 2014-05-06
RFC PR: https://github.com/emberjs/rfcs/pull/50
Ember Issue: (leave this empty)

---

# Summary

The `{{action` helper should be improved to allow for the creation of
closed over functions that can be passed between components and passed
the action handlers.

See [this example JSBin from @rwjblue](http://emberjs.jsbin.com/rwjblue/466/edit?html,js,output)
for a demonstration of some of these ideas.

# Motivation

Block params allow data to be passed from one component to a downstream
component, however there is currently no way to pass a callback to a downstream
component.

# Detailed design

First, the existing uses of `{{action` will be maintained. An action can be attached to an
element by using the helper in element space:

```hbs
{{! app/index/template.hbs }}
{{! submit action will hit immediate parent }}
<button {{action "submit"}}>Save</button>
```

An action can be passed to a component as a string:

```hbs
{{! app/index/template.hbs }}
{{my-button on-click="submit"}}
```

```js
// app/components/my-button/component.js
export default Ember.Component.extend({
  click: function(){
    this.sendAction('on-click');
  }
});
```

Or a default action can be passed:

```hbs
{{! app/index/template.hbs }}
{{my-button action="submit"}}
```

```js
// app/components/my-button/component.js
export default Ember.Component.extend({
  click: function(){
    this.sendAction();
  }
});
```

In all these cases, `submit` is called on the parent context relative to the scope `action` is
attached in. The value `"submit"` is attached to the component in the last two as
`this.attrs.on-click` or `this.attrs.action`, although it is not directly used.

### Creating closure actions

Closure actions are created in a template and may be used in all places a string
action name can be used. For example, this current functionality:

```hbs
<button {{action "submit" on="click"}}>Save</button>
```

Would be written using a closure action as:

```hbs
<button {{action (action "submit") on="click"}}>Save</button>
```

The functionality is exactly the same as the string-based action example.
How does that happen?

* `(action "submit")` reads the `submit` function off the current scope's
  `actions.submit` property.
* It then creates a closure to call that function.
* `{{action` receives that function as a param. It registers a listener (in
  this case on click) and when fired calls the closure function.

Consider usage on the calling side. With the current string-based actions:

```hbs
{{my-component action="submit"}}
```

```js
export default Ember.Component.extend({
  click: function(){
    this.sendAction(); // submit action, legacy
    // this.attrs.action is a string
    this.attrs.action; // => "submit"
  }
});
```

With closure actions, the action is available to call directly. The `(action` helper
wraps the action in the current context and returns a function:

```hbs
{{my-component action=(action "submit")}}
```

```js
export default Ember.Component.extend({
  click: function(){
    this.sendAction(); // submit action, legacy
    // this.attrs.action is a function
    this.attrs.action(); // submit action, new style
  }
});
```

A more complete example follows, with a controller for context:

```js
// app/index/controller.js
export default Ember.Controller.extend({
  actions: {
    submit: function(){
      // some submission task
    }
  }
});
```

```hbs
{{! app/index/template.hbs }}
{{my-button save=(action 'submit')}}
```

```js
// app/components/my-button/component.js
export default Ember.Component.extend({
  click: function(){
    this.attrs.save();
    // for backwards compat, you may also this.sendAction('save');
  }
});
```

### Hole punching with a closure-based action

The current system of action bubbling falls down quickly when you want to pass a message through multiple
levels of components. A closure based action system helps address this.

Instead of relying on bubbling, a closure action wraps an action from the current context's
`actions` hash in a function that will call it on that context. For example:

```hbs
{{! app/index/template.hbs }}
{{my-form submit=(action 'submit')}}
```

```hbs
{{! app/components/my-form/template.hbs }}
{{my-button on-click=attrs.submit}}
```

```hbs
{{! app/components/my-button/template.hbs }}
<button></button>
```

```js
// app/components/my-button/component.js
export default Ember.Component.extend({
  click: function(){
    this.attrs['on-click']();
    // for backwards compat, you may also this.sendAction();
  }
});
```

A closure action can also be called by an action handler:

```hbs
{{! app/index/template.hbs }}
{{my-form submit=(action 'submit')}}
```

```hbs
{{! app/components/my-form/template.hbs }}
{{my-button on-click=submit}}
```

```hbs
{{! app/components/my-button/template.hbs }}
<button {{action on-click}}></button>
```

Lastly, closure actions allow for yielding an action to a block. For example:

```hbs
{{! app/index/template.hbs }}
{{my-form save=(action 'submit') as |submit reset|}}
  <button {{action submit}}>Save</button>
  {{! ^ goes to my-form's save attr property, which
        is the submit action on the outer scope }}
  <button {{action reset}}>Reset</button>
  {{! ^ goes to my-form }}
  <button {{action "cancel"}}>Cancel</button>
  {{! ^ goes to outer scope }}
{{/my-form}}
```

```hbs
{{! app/components/my-form/template.hbs }}
{{yield attrs.save (action 'reset')}}
```

```js
// app/components/my-form/component.js
export default Ember.Component.extend({
  actions: {
    reset: function(){
      // rollback
    }
  }
});
```

### Currying arguments with a closure-based action

With string-based actions, an argument can be passed to the called function. For
example:

```hbs
<button {{action "save" model}}></button>
```

```js
export default Ember.Component.extend({
  actions: {
    save: function(model) {
      model.save();
    }
  }
});
```

Closure actions allow for another opportunity to curry arguments. Arguments
set by an element action helper simply add to the end of the arguments list:

```hbs
{{! app/index/template.hbs }}
{{my-component save=(action "save" model)}}
```

```hbs
{{! app/components/my-component/template.hbs }}
<button {{action attrs.save prefs}}></button>
```

```js
// app/index/controller.js
export default Ember.Controller.extend({
  actions: {
    save: function(model, prefs) {
      model.set('prefs', prefs);
      model.save();
    }
  }
});
```

Multiple arguments can be curried or set at any level. If an action is called ala
`this.attrs.save(additionalPrefs)`, that final argument is added
to the end of the arguments list.

### Re-targeting the scope of a closure action

The `target` option may be provided to specify what scope the closure is called
with. For example:

```hbs
{{! app/index/template.hbs }}
<my-component on-click={{action "save" model target=someComponentInstance}}></my-component>
```

Much like with the `{{action` helper, passing both a
target and a bound argument will throw.

The default target for a closure is always the current scope.

* When routable components land, the current component will be the default target.
* If a controller is the current scope, that controller will also be a default target.
* A route will *never* be a closure action target. String actions will continue
  to have their current behavior of bubbling to the route.

A later proposal will determine how actions on a route are passed to a routable
component.

### Return values of a closure action

Closure actions return the returned value of their called function. For example:

```js
// app/index/controller.js
export default Ember.Controller.extend({
  actions: {
    submit: function(){
      return 'great success';
    }
  }
});
```

```hbs
{{! app/index/template.hbs }}
{{my-button save=(action 'submit')}}
```

```js
// app/components/my-button/component.js
export default Ember.Component.extend({
  click: function(){
    var result = this.attrs.save();
    // for backwards compat, you may also this.sendAction('save') but
    // in that case you do not have access to the return value.
    result; // => 'great success'
  }
});
```

### Actionable object with INVOKE

`{{mut` is a new helper in Ember.js. It is not yet widely used in Ember apps, but its
interaction with the action helper is important to align early on.

Mut objects represent a modifiable value. For example with tag-based components:

```hbs
{{! app/index/template.hbs }}
<my-form name={{mut model.name}}></my-component>
```

This will cause a mutable property to be added to `attrs`. To update the name,
`this.attrs.name.update(newName)` can be called. The value can be read (in
JavaScript) as `this.attrs.name.value`.

Often, a mutable value will be set as the result of an action. Mutable values
can be called actionable. For example:

```hbs
{{! app/index/template.hbs }}
<my-form submit={{action (mut model.name)}}></my-component>
```

```js
// app/components/my-form/component.js
export default Ember.Component.extend({
  click() {
    const value = this.get('newValue');
    this.attrs.submit(value);
  }
});
```

What is happening here?

* `(mut model.name)` creates a mutable object for the `model.name` value.
* `{{action (mut model.name)}}` tests the passed object for a property with the
  key `INVOKE` (an internal symbol). This value is a function that updates the mutable value.
* Action wraps the calling of the `INVOKE` property in a function like any
  other action, and passes it to the `attrs`.

Thus, when the action is called the argument is passed to `INVOKE` which uses
it to update the mutable value. This is a simple way to enable the "actions up"
part of component-driven app architecture without ceremony around changing state.

### Plucking a property from the first argument with value

A component (or when Ember supports this better, an element) may emit an event
object and pass it to an action. In this case the value will need to be read off
the event before it can be passed to the action function. For example:

```hbs
{{input input=(action 'setName')}}
```

```js
export default Ember.Component.extend({
  actions: {
    setName(event) {
      this.get('model').set('name', event.target.value);
    }
  }
});
```

The action serves only to read the value off of the event. Here the `value`
option can be used as sugar to accomplish the same task:

```hbs
{{input input=(action (mut model.name) value="target.value")}}
```

The `value` path is read off of whatever the first argument to the actions is.

* `(mut model.name)` becomes a function, our action
* When the `input` event fires, the function is called with the event as the
  first argument.
* The first argument is re-written to the value of `event.target.value`
* The function wrapping the `mut` is set
* The `mut` is updated.

This option is designed to align with future plans for `on-some-event` handlers
for html elements.

# Drawbacks

Currently `{{action` is only used in an element space:

```hbs
<button {{action "booyah"}}>Fire</button>
```

The closure usage is a new, perhaps `action` is not the right word. However the two
behaviors are pretty similar in their conceptual behavior.

* `{{action` in element space attaches an event listener that fires a bubbling
  action.
* `(action` closes over an action from the current scope so it can be attached
  via `{{action` or passed around and called later.

This confusion should go away as we move to an `on-click` event listener pattern,
ala `<button on-click={{someClosureAction}}>`.

Additionally, there may be developers who still have `{{action someActionName}}` instead
of the quoted version. This is long deprecated, but these apps may see some
unexpected behavior.

Also additionally, some emergent behaviors exist that may not be desired as real APIs. For example,
an action being a function means it can be passed directly to event handlers:

```hbs
{{my-component mouseEnter=(action 'didEnter')}}
```

The actual API we plan for 2.0 (ideally) is:

```hbs
{{my-component on-mouse-enter=(action 'didEnter')}}
```

These behaviors should not be documented, and we should make clear that they rely on behavior that
will be deprecated. A mitigating move is to *not* proxy actions through to
`get` on a component, and only allow them to be accessed on `attrs`.

Lastly, default actions may look a bit confusing:

```hbs
{{my-button action=(action 'action')}}
{{! ^ this is valid }}
```

But the quoted string syntax is not being removed.

# Alternatives

There is maybe a thing called `ref` that solves this same problem. There has also
been discussion of accessing properties on `outlet` across all child components
and their layouts, which would allow easy targetting of the top level component.

# Unresolved questions

Interaction with `ref` or `outlet.` if any..
