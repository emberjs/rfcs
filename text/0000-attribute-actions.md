- Start Date: 2015-10-17
- RFC PR: https://github.com/emberjs/rfcs/pull/100
- Ember Issue: (leave this empty)

# Summary

Ember's `action` helper has two inconsistent forms. The goal of this RFC is to
allow the deprecation or removal of the classic action helper by formalizing
the DOM event listener API. This API is already seeing adoption with closure
actions.

Because this API is already seeing informal adoption, the design changes here
are very minimal.

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
  * Only functions should be permitted as a bound value for `onclick=`. For
    example `<button onclick={{someString}}>` should assert.
* In Ember 2.x+1, classic actions like `<button {{action 'save' on='click'}}>`
  will be deprecated.
* In Ember 3.0, classic actions will be removed.

# Motivation

Classic actions bubble an action name through handlers, and declared in the
"element space" of an element. For example:

```hbs
<button {{action 'save'}}></button>
```

Closure actions pluck a function from the immediate context of a template,
and return another function closing over that function and any arguments. These
are used where subexpressions are valid in Ember, for example:

```hbs
{{my-button onChange=(action 'save')}}
```

or

```hbs
<button onclick={{action 'save'}}>
```

Knowing when one style should be used over another confuses Ember developers.

Each of the two styles has a different handler implementation.
Return values from closure actions are treated as return
values to the invocation site, but classic actions use the return value to
continue or cancel bubbling.

Invoking an action in JavaScript differs depending on the object you
invoke it from. For example to call an
action from a component `sendAction('actionName')` is called. To call an action
from a controller (and bubble it to the route), `send('actionName')` is used.

By removing the remaining use-cases for classic actions, nearly all these
confusing discrepancies can be eliminated. Ember is easier to learn and master.

# Detailed design

All attributes of a DOM node named `on*` will be considered attribute actions.
This will be done as a wildcard match: `onclick=`, `omouseover=`, and `only=`
would all be considered event listeners.

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
have the advantage of a clear and extensible syntax. Consider the simplicty of
attaching a `mouseover` event with attribute actions:

```hbs
<div onmouseover={{action 'save'}}>Save</div>
```

The `onmouseover` attribute action is an natural evolution of the
orignal `onclick` usage.

A second advantage this change is its parity with Ember component
action passing. For example consider these components:

```hbs
<button onclick={{action 'save'}}>Save</button>
<my-button save={{action 'save'}}>Save</my-button>
{{my-button save=(action 'save')}}Save{{/my-button}}
```

Action passing to a component is visually similar to the new
attribute attachment syntax. Or course an `on*` attribute passed to an
`Ember.Component` is simply a function with no special behavior.

Passing non-actions to `on*` attributes will be discouraged.

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
`addEventListener`. This ensures that if there are multiple handlers making
their way onto an element they do not stomp on each other. Any attribute named
`on*` will be attached as such.

#### Web Components

This RFC suggests official support for actions on Web Components be added to
Ember.

The web component [best practices](http://webcomponents.org/articles/web-components-best-practices/)
doc as well as discussion on
[2ality](http://www.2ality.com/2015/08/web-component-status.html)
agree that data coming out of a web component should use events.

The syntax for attaching a listener to a web component will be the same as
any event:

```hbs
<my-custom oncustomfoo={{action someFunction}}></my-custom>
```

Merely setting the `oncustomfoo` property on the DOM node is not sufficient
([this jsbin](http://jsbin.com/vedahareda/1/edit?html,js,output) demonstrates
custom events not calling functions on properties).
The implementation of `my-custom` may (and should) dispatch events to send out
data.

This is partial motivation for using `addEventListener` over setting the
function on a prop.

#### Glimmer Components

Glimmer components may (and this is speculation more than design) permit
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
attached. Using the rules we have thus far for property reflection and
Glimmer Components, these props would instead smash. The natural merge
semantics of `addEventListener` are a second
motivation for using that API over setting a function on a prop.

# Drawbacks

#### Order of execution

Other events managed by Ember use a bespoke solution which delegates to
handlers from a single listener on the document body. Because this manager
is on the body, an event dispatched by that system will always execute after
all attribute actions.

[This JSBin](http://emberjs.jsbin.com/refanumatu/1/edit?html,js,output) demonstrates using action and onclick, and you can see that the
order of the wrapper click event and the action/onclick attached function
are not consistent. Events that will still execute via the event manager are:

* Those on `Ember.Component` components
* Those attached via element space `{{action 'foo' on="click"}}` usage
* Possibly `Ember.GlimmerComponent` components, however these are current
  still un-designed in this area.

#### Inconsistency with legacy event names

Ember's [current event naming schema](http://emberjs.com/api/classes/Ember.View.html#toc_event-names)
for handlers on components and to the `on=` argument of `{{action}}` does not
stay consistantly lowercase as the native attribute names do. This inconsistancy
is unfortunate, but moving to the native event names is less surprising than
maintaining the proprietary capitalization.

# Alternatives

#### Wildcard on\*

Considered (and rejected) was the idea of using `on-*` instead of the native
action attribute name. This option would make clear that native semantics do
not apply, and we could perhaps opt-in to some consistency features such as
delegating attribute actions from the event manager.

By treating all DOM attributes with `on*` as attribute actions, we stomp on
several english language words that would no longer be viable attribute names.
The [list of words is not terrible](http://www.morewords.com/starts-with/on/).

A whitelist is not appropriate, as web components may emit custom event names
we have not expected. Some other options to consider:

* Fall back to `on-*` as an official syntax.
* Use a runtime-configurable whitelist of attribute action event names.
* Use a runtime-configurable blacklist of attribute action event names.

# Unresolved questions

None.
