- Start Date: 2015-10-17
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Ember's `action` helper has two inconsistent forms. The goal of this RFC is to
allow the deprecation or removal of the classic action helper by adding a new
DOM event listener API designed for closure actions.

The RFC suggests three phases of implementation:

* In Ember 2.x, kebab actions are released as an API
* In Ember 2.x+1, classic actions will be deprecated
* In Ember 3.0, classic actions will be removed

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

#### Kebab Action Attachment

To add an event listener to a DOM node, the author may use "kebab" syntax. By
this we mean the dasherized form of an event name, prefixed by `on-`.

For example to attach a `click` event to a button, the following syntax would
be used:

```hbs
<button on-click={{action 'save'}}>Save</button>
```

For comparison, to attach the `save` action with classic actions this syntax
would be used instead:

```hbs
<button {{action 'save'}}>Save</button>
```

Kebabs are more verbose in the above case, but have the
advantage of a clear and extensible syntax. Consider attaching a
`mouseover` event:

```hbs
<div on-mouse-over={{action 'save'}}>Save</div>
```

vs.

```hbs
<div {{action 'save' on='mouseOver'}}>Save</div>
```

The kebab version is an basic evolution of the `on-click` usage.

The second advantage of kebab syntax is its parity with Ember component
action passing. For example consider these components:

```hbs
{{my-button on-click=(action 'save')}}Save{{/my-button}}
<my-button on-click={{action 'save'}}>Save</my-button>
```

Kebab syntax is much closer to the above than classic action syntax. Of course
a kebab action passed to a component (or angle bracket component) is simply
a function with no special behavior.

If any non-function is passed to a kebab, such as in
`on-click="alert('blah');"` an assertion should fail. Only functions are
permitted.

#### Handling a Kebab Action

Kebab actions will be invoked with the raw DOM event (not the jQuery event)
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
<button on-click={{action 'save'}}>Save</button>
```

Actions curry arguments, thus the event may not always be the first argument. For
example:

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
<input on-change={{action 'log' 'value is:'}} />
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
<input on-change={{action 'log' value='target.value'}} />
```

Passing the event is useful for interacting with the originating DOM node,
but additionally important for allowing event propagation to be controlled
(cancelled).

#### Event Management

Kebab actions are lazy. If a kebab action of `on-flummux` is used, then Ember
should listen for the event of `flummux` on the root element and dispatch
that action when it fires.

If an event from the [list of events Ember listens for](http://emberjs.com/api/classes/Ember.View.html#toc_event-names)
is used, then kebabs can use the already existing listeners.

# Drawbacks

Obviously, if this is an RFC, there cannot be drawbacks.

j/k.

Kebab actions, as described here, use a dasherized multi-word format. For
example `on-mouse-over`. This corresponds to how Ember already camelizes
event listeners (for example `mouseOver` is the method for a component
listener), but does *not* correspond to the native browser APIs which would
imply `on-mouseover`. As what form of camelization, dasherization, or single-word-ness
to use can be extremely confusing to a newcomer, I am inclined to suggest
matching the native browser instead of following Ember's convention of splitting
the words.

# Alternatives

One immediate alternative making the rounds today is using event listeners
directly. For example:

```hbs
<button onclick={{action 'save'}}></button>
```

The above passes a function to the `onclick` property of the `<button>` DOM
node. The upside of recommending this usage is the directness of it: You can
demonstrate what is going on trivially. However the downside is that many
more event listeners are created than if we instead used the Ember event
manager and share listeners across the DOM.

# Unresolved questions

At various times, "auto-capitalization" has been suggested. This would mean
a property passed as `on-click` would also be available on the `attrs` of
a component would be available as `onClick`. This is not directly related
this this RFC, but is in the same universe of actions related things and
could be suggested here. There are downsides to camelization however, such
as duplicating each kebab attr passed.

Can this strategy be used as an opt-in for capture events? I'm unsure it
can, since it must interact with the rest of Ember's event system (currently
bubbling based on jQuery).
