---
Start Date: 2019-03-21
Relevant Team(s): Ember.js, Learning
RFC PR: https://github.com/emberjs/rfcs/pull/471
Tracking: https://github.com/emberjs/rfc-tracking/issues/32

---

# `{{on}}` Modifier

## Summary

Add an `{{on}}` modifier for adding event listeners to elements.

## Motivation

Currently, there are two ways to bind event listeners to elements in Ember
templates with built-in, official Ember APIs:

1. Use the `{{action}}` element modifier
2. Use the `on*=` property bindings

Both of these solutions are problematic for a number of reasons.

The `{{action}}` modifier:

- Uses a non-standard AST transform to pass `this` in order to bind the context
  of the template (the component) to the function (typically a component
  method). This is problematic behavior, not just because of the transform, but
  because of the implicit state that is being passed to the modifier in the
  first place, and the fact that the modifier is "magical" in the sense that it
  can do things that no other modifier or helper can do.
- Binds to the `click` event by default, which requires a bit of knowledge to
  understand on first glance, and requires a bit of extra knowledge to figure
  out how to bind to other events. Compared to `onclick=`, this is less
  explicit, and requires more knowledge of the framework out of the box.
- Is confusing when used with the `@action` decorator to bind the function, and
  the `{{action}}` helper. The term "action" is highly overloaded here, and it
  is unclear what it concretely means without context.

The `on*` properties, by contrast, may seem to be a better approach overall to
the problem, but they have their own issues:

- The require the ability to bind to element _properties_. Properties are _not_
  attributes, and the distinction is subtle, but important. Most importantly,
  props are _not_ rendered, and thus are _not_ SSR/rehydration friendly, since
  it is difficult to assign during rehydration. By constrast, modifiers are
  specifically designed around SSR. They do not trigger at all during server
  side render, and then are run specifically during rehydration.
- Not all events are bindable via `on*` properties, and some events such as
  `focus` have different behavior than when assigned with `addEventListener`
  directly. This behavior has been a pain point in other frontend frameworks,
  and requires workarounds in most cases.
- They do not work on certain elements at all, such as `<svg>`.
- They are not compatible at all with standard web components. Web components
  don't send events in the same way as standard elements for properties, and
  instead rely on event listeners added via `addEventListener`. While web
  components are not currently used in the Ember ecosystem, they are advancing
  and may eventually be useful for smaller, self-contained components,
  especially for users that want to share code between framework ecosystems.

Neither of these solutions is perfect for solving the problem of adding event
listeners. This RFC proposes adding a new modifier to Ember: the `{{on}}`
modifier. This modifier will explicitly add event listeners using the
`addEventListener` API.

## Credits

The design of this modifier is based on [ember-on-modifier](https://github.com/buschtoens/ember-on-modifier)
by Jan Buschtöns, which is an excellent addon that has allowed us to test this
design in real apps and get feedback about the design.

## Detailed design

The `{{on}}` modifier will recieve:

1. The event name as a string as the first positional parameter
2. The event listener function as the second positional parameter
3. Named parameters as options

The following usages are equivalent:

```hbs
<div {{on "click" this.handleClick passive=true}}></div>
```

```js
element.addEventListener('click', this.handleClick, { passive: true });
```

Multiple event listeners can be added to an element by adding the modifier more
than once:

```hbs
<div
  {{on "click" this.handleClick}}
  {{on "mouseenter" this.handleMouseEnter}}
>
</div>
```

Extra positional parameters will be ignored. In order to pass values to a
function, users should use a helper such as the [fn helper](https://github.com/emberjs/rfcs/pull/470):

```hbs
<div {{on "click" (fn this.addNumber 123)}}></div>
```

The event will also be passed as a parameter to the function, so it can be used
directly.

`{{on}}` does _not_ bind context. Context binding must be handled via a helper,
or in the component definition via the `@action` decorator, which will be the
recommended path:

```js
class ClickComponent extends Component {
  @action
  handleClick() {
    // ...
  }
}
```

### Named Parameters

The following named parameters will be accepted as options to `{{on}}`:

- `capture`
- `once`
- `passive`

These will be passed forward to `addEventListener` as options in modern
browsers, and will be polyfilled in older browsers. Other options will be
ignored.

## How we teach this

`{{on}}` itself can be taught in conjuction with event listeners. We can give
users an overview of event listeners and the `addEventListener` API, and then
relate the modifier directly to it. We should provide lots of examples of usage
throughout the guides for how users can use it, including some examples of how
to apply multiple event listeners to an element.

`{{on}}` also does not bind context, and we should make it clear that the
`@action` decorator is the recommended way to bind methods to their components.

Other than that, most examples in the guides can be translated directly from
the `{{action}}` modifier to the `{{on}}` modifier.
Documentation work would include:

- revising [Handling Events](https://guides.emberjs.com/release/components/handling-events/#toc_event-names) or the corresponding Octane article in entirety
- revising [Triggering Changes with Actions](https://guides.emberjs.com/release/components/triggering-changes-with-actions/) or the corresponding Octane article
- Updating 50+ uses of `{{action}}` in the guides
- Adding `{{on}}` as a new API docs entry
- Update examples within the API docs to show all uses of actions that are supported in the API docs - `{{on}}`, `{{action}}`, `onclick={{action}}`

`{{action}}` is used 130+ times in the API docs. Not all examples would need to be updated, however an audit would be needed to figure out which need to be added to/changed.

## Drawbacks

- This API is somewhat more verbose than the `{{action}}` modifier since it
  requires a helper for passing additional values to the function. However, this
  explicitness makes it much more clear what the separation of concerns is, and
  allows users to fine tune their event handling.
- The `{{on}}` modifier requires another helper to pass values. Currently, the
  ideal companion helper is the `{{fn}}` helper, which is still in RFC.
- The `onclick=` style at first looks like a native browser API, and is
  sometimes easier to teach because of this. However, it is _not_ the same as
  the `onclick` attribute, which only receives strings, and the differences are
  often confusing. In addition, the fact that there are many use cases that
  aren't covered by these properties, and the fact that they are hostile to SSR,
  makes them less than ideal.

## Alternatives

One option would be to allow the `{{on}}` modifier to receive event listeners
as named parameters instead:

```hbs
<div
  {{on
    click=this.handleClick
    mouseenter=this.handleMouseEnter
  }}
>
</div>
```

One downside of this style of API is that there is no clear way to pass
additional options to the individual event listeners. We could pass them as a
positional parameter, but they would apply to _all_ listeners, which would
require multiple invocations.

We could also allow the `{{on}}` modifier to bind context. This opens up a
question: Should helpers and modifiers by able to use the context of the
template implicitly? Since helpers are analogous to functions in JavaScript,
this would be an equivalent API choice in JS:

```js
function foo() {
  this.bar = 123;
}

class Foo {
  doSomething() {
    foo();
  }
}

let foo = new Foo();

foo.doSomething();
console.log(foo.bar); // 123
```

Arbitrary functions being able to access `this` when used within a class like so
seems problematic from a language design standpoint, and we believe an API like
this in Ember's templates would be very problematic, and easy to misuse.
