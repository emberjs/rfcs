- Start Date: 2015-10-17
- RFC PR: https://github.com/emberjs/rfcs/pull/100
- Ember Issue: (leave this empty)

# Summary

Ember's `action` helper has two inconsistent forms. The goal of this RFC is to
allow the deprecation or removal of the classic action helper by adding a new
DOM event listener API designed for closure actions.

The RFC suggests three phases of implementation:

* In Ember 2.x (after this RFC is merged), attribute actions like
  `<button onclick={{action 'save'}}>` are declared an
  official API.
  * There are some very small code changes, but largely this is a
    messaging and documentation change.
  * These "attribute events" should use `element.addEventListener(eventName, handler, false)`
    to attach instead of `element['on'+eventName'] = handler` for attachment.
  * Non-action functions should warn if they are passed to `onclick=`
    APIs.
  * Non-functions should not be permitted as a bound value for `onclick=`, for
    example `<button onclick={{someString}}>`. This should assert.
* In Ember 2.x+1, classic actions like `<button {{action 'save' on='click'}}>`
  will be deprecated.
* In Ember 3.0, classic actions will be removed.

Additionally it suggests:

* No new APIs (such as glimmer components) will leverage the event manager
  currently used in Ember. Specifically, it is recomended that glimmer
  components not implement any support for `click() {` handlers and they
  instead rely on attaching events via the template: `<root-element onclick={{action 'save'}}>`

# Motivation

Classic actions bubble a string action through handlers, and are used in the
space of an element. For example: `<button {{action 'save'}}></button>`.

Closure actions pluck a function from the immediate context of a template,
and return another function closing over that function and arguments. These
are used where subexpressions are valid in Ember, for example
`{{my-button onChange=(action 'save')}}` or
`<button onclick={{action 'save'}}>`.

The differences between these two usages leak into the handler implementation
as well. While returned values from closure actions are treated as return
values to the invocation site, classic actions use the return value to continue
or cancel bubbling.

Furthermore, even invoking an action can be inconsistent. For example to call an
action from a component `sendAction('actionName')` is called. To call an action
from a controller (and bubble it to the route), `send('actionName')` is used.

By removing the remaining use-case for classic actions, nearly all these
confusing discrepancies can be eliminated. Ember is easier to learn and master.

# Detailed design

#### Action Attachment

To add an event listener to a DOM node, the author may use attributes with
the value of an action.

For example to attach a `click` event to a button, the following
"attribute action" syntax would be used:

```hbs
<button onclick={{action 'save'}}>Save</button>
```

For comparison, to attach the `save` action with classic actions this syntax
is used today:

```hbs
<button {{action 'save'}}>Save</button>
```

Attribute actions are more verbose in the above case, but
have the advantage of a clear and extensible syntax. Consider attaching a
`mouseover` event:

```hbs
<div onmouseover={{action 'save'}}>Save</div>
```

vs.

```hbs
<div {{action 'save' on='mouseOver'}}>Save</div>
```

The `onmouseover` attribute action is an evolution of the `onclick` usage.

A second advantage this change is its parity with Ember component
action passing. For example consider these components:

```hbs
{{my-button save=(action 'save')}}Save{{/my-button}}
<my-button save={{action 'save'}}>Save</my-button>
```

This common usage of action passing is visually very similar to the new
attribute attachment syntax. Or course an `on*` attribute passed to an
`Ember.Component` is simply a function with no special behavior.

Passing non-actions to `on*` attributes will be discouraged.

* Literals should be permitted, such as `onclick="window.alert('foo')"`.
* Non-function bindings are not permitted, and will throw from an assertion.
  For example `onclick={{someBoundString}}` will cause an assertion to throw.
* Non-action functions such as `onclick={{someBoundFunction}}` should cause
  a warning to be logged. This warning will encourage the user to use
  `onclick={{action someBoundFunction}}` or use actions as normally
  documented.

#### Handling an Attribute Action

`on*` attributes will be invoked with the raw DOM event (not the jQuery event)
as an argument. For example:

```js
// app/components/my-button.js
export Ember.Component.extend({
  actions: {
    save(event) {
      console.log('saved with click on ', event.target);
      this.get('model').save();
    }
  }
});
```

```hbs
{{! app/templates/components/my-button.hbs }}
<button onclick={{action 'save'}}>Save</button>
```

Actions create a closure over arguments, thus the event may not always be
the first argument. For example:

```js
// app/components/my-input.js
export Ember.Component.extend({
  actions: {
    log(prefix, event) {
      console.log(prefix, Ember.$(event.target).val());
    }
  }
});
```

```hbs
{{! app/templates/components/my-input.hbs }}
<input onchange={{action 'log' 'value is:'}} />
```

Additionally, the `value` argument to `action` is useful when passing the
event.


```js
// app/components/my-input.js
export Ember.Component.extend({
  actions: {
    log(value) {
      console.log(target);
    }
  }
});
```

```hbs
{{! app/templates/components/my-input.hbs }}
<input onchange={{action 'log' value='target.value'}} />
```

Passing the event is useful for interacting with the originating DOM node,
but additionally important for allowing event propagation to be controlled
(cancelled).

#### Event Dispatching

Attribute actions utilize a native API for dispatching that places them on
the bubbling event dispatcher. Browsers supported by Ember 2.x have three
kind of DOM event attachment:

* `el.addEventListener('click', handlerFn, false)` add a handler to the bubbling
  phase.
* `el.addEventListener('click', handlerFn, true)` add a handler to the capture
  phase.
* `el.onclick = handlerFn;` add a handler to the bubbling phase.

Attribute actions will be dispatched on the bubbling phase, and attached via
`addEventListener`. This ensures that if they are multiple handler making
their way onto an element they do not stomp on each other.

Glimmer components may (and this is specilation more than design) permit
multiple events to be attached via reflection of invocation attrs to the
root element. For example:

```hbs
{{! app/routes/index/template.hbs }}
{{! Invoke the component 'my-button' }}
<my-button onclick={{action 'save' model}}>
```

```hbs
{{! app/components/my-button/template.hbs }}
{{! Create a div for the root element, with a logging action attached }}
<div onclick={{action 'logClick'}}>
```

The `logClick` handler should be attached prior to the `save` handler being
attached.

# Drawbacks

Other events managed by Ember use a bespoke solution which delegates to
handlers from a single listener on the document body. Because this manager
is on the body, an event dispatched by that system will always execute after
all attribute actions.

[This JSBin](http://emberjs.jsbin.com/refanumatu/1/edit?html,js,output) demonstrates using action and onclick, and you can see that the
order of the wrapper click event and the action/onclick attached function
are not consistent. Events that will still execute via the event manager are:

* Those on `Ember.Component` components
* Those attached via element space `{{action 'foo' on="click"}}` usage

This RFC schedules the deprecation the second API, however to ensure the
ordering problems of `Ember.Component` do not continue into glimmer components
it also recommends that `Ember.GlimmerComponent` have no `save() {` event
attachment API. Instead attribute actions on the root element should be
used.

Additionally, Ember's [current event naming schema](http://emberjs.com/api/classes/Ember.View.html#toc_event-names)
for handlers on components and to the `on=` argument of `{{action}}` does not
stay consistantly lowercase as the native attribute names do. This inconsistancy
is unfortunate.

# Alternatives

Considered (and rejected) was the idea of using `on-*` instead of the native
action attribute name. This option would make clear that native semantics do
not apply, and we could perhaps opt-in to some consistency features such as
delegating attribute actions from the event manager.

# Unresolved questions

There is some debate over if `Ember.GlimmerComponent` event attachment should
skip the event manager or not. This has unknown implications to the plan here.
If it should use the event manager, then what happens to ordering of event
handler execution? Do we return to all actions being on the event manager?
