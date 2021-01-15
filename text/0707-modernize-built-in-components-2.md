---
Stage: Accepted
Start Date: 2021-01-14
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/707
---

# Reduce API Surface of Built-In Components

## Summary

In order to reduce the API surface of the built-in `<LinkTo>`, `<Input>` and
`<Textarea>` components, we propose to deprecate all named arguments on these
components *except* the following:

* `<LinkTo>`
  * `@route`
  * `@model`
  * `@models`
  * `@query`
  * `@replace`
  * `@preventDefault`
  * `@disabled`
  * `@current-when`
  * `@activeClass`
  * `@loadingClass`
  * `@disabledClass`
* `<Input>`
  * `@type`
  * `@value`
  * `@checked`
  * `@insert-newline`
  * `@enter`
  * `@escape-press`
* `<Textarea>`
  * `@value`
  * `@insert-newline`
  * `@enter`
  * `@escape-press`

## Motivation

This is a follow-up to [RFC #671](./0671-modernize-built-in-components-1.md)
and shares the same high-level motivations and historical context.

## Detailed design

The following named arguments should be explicitly documented as public:

* `<LinkTo>`
  * `@route`
  * `@model`
  * `@models`
  * `@query`
  * `@replace`
  * `@preventDefault`
  * `@disabled`
  * `@current-when`
  * `@activeClass`
  * `@loadingClass`
  * `@disabledClass`
* `<Input>`
  * `@type`
  * `@value`
  * `@checked`
  * `@insert-newline`
  * `@enter`
  * `@escape-press`
* `<Textarea>`
  * `@value`
  * `@insert-newline`
  * `@enter`
  * `@escape-press`

The arguments not enumerated above are either no longer necessary, recommended
or accidentally exposed private implementation details.

### No Longer Necessary

#### HTML Attributes

The built-in components historically accepted a varierty of named arguments for
applying certain HTML attributes to the component's HTML element. This includes
the following (may not be a complete list):

* `<LinkTo>`
  * `@id`
  * `@elementId` (alias for `@id`)
  * `@ariaRole` (maps to the `role` HTML attribute)
  * `@class`
  * `@classNames` (deprecated, expands into the `class` HTML atttribute)
  * `@classNameBindings` (deprecated, expands to the `class` HTML atttribute)
  * `@isVisible` (deprecated, expands to the `display: none` inline style)
  * `@rel`
  * `@tabindex`
  * `@target`
  * `@title`
* `<Input>`
  * `@id`
  * `@elementId` (alias for `@id`)
  * `@ariaRole` (maps to the `role` HTML attribute)
  * `@class`
  * `@classNames` (deprecated, expands into the `class` HTML atttribute)
  * `@classNameBindings` (deprecated, expands to the `class` HTML atttribute)
  * `@isVisible` (deprecated, expands to the `display: none` inline style)
  * `@accept`
  * `@autocapitalize`
  * `@autocomplete`
  * `@autocorrect`
  * `@autofocus`
  * `@autosave`
  * `@dir`
  * `@disabled`
  * `@form`
  * `@formaction`
  * `@formenctype`
  * `@formmethod`
  * `@formnovalidate`
  * `@formtarget`
  * `@height`
  * `@indeterminate`
  * `@inputmode`
  * `@lang`
  * `@list`
  * `@max`
  * `@maxlength`
  * `@min`
  * `@minlength`
  * `@multiple`
  * `@name`
  * `@pattern`
  * `@placeholder`
  * `@readonly`
  * `@required`
  * `@selectionDirection`
  * `@size`
  * `@spellcheck`
  * `@step`
  * `@tabindex`
  * `@title`
  * `@width`
* `<Textarea>`
  * `@id`
  * `@elementId` (alias for `@id`)
  * `@ariaRole` (maps to the `role` HTML attribute)
  * `@class`
  * `@classNames` (deprecated, expands into the `class` HTML atttribute)
  * `@classNameBindings` (deprecated, expands to the `class` HTML atttribute)
  * `@isVisible` (deprecated, expands to the `display: none` inline style)
  * `@autocapitalize`
  * `@autocomplete`
  * `@autocorrect`
  * `@autofocus`
  * `@cols`
  * `@dir`
  * `@disabled`
  * `@form`
  * `@lang`
  * `@maxlength`
  * `@minlength`
  * `@name`
  * `@placeholder`
  * `@readonly`
  * `@required`
  * `@rows`
  * `@selectionDirection`
  * `@selectionEnd`
  * `@selectionStart`
  * `@spellcheck`
  * `@tabindex`
  * `@title`
  * `@wrap`

These arguments are no longer necessary – with angle bracket invocations, HTML
attributes can be passed directly. An invocation passing one or more of these
named arguments shall trigger a deprecation warning simialr to this:

```
<Input @placeholder="Ember.js" />
       ~~~~~~~~~~~~~~~~~~~~~~~
or

{{input placeholder="Ember.js"}}
        ~~~~~~~~~~~~~~~~~~~~~~

Passing the `@placeholder` argument to <Input> is deprecated. Instead, please
pass the attribute directly, i.e. `<Input placeholder={{...}} />` instead of
`<Input @placeholder={{...}} />` or `{{input placeholder=...}}`.
```

A notable exception when passing an argument named `@href` to the `<LinkTo>`
component. This was never intentionally supported and will trigger an error
instead of a deprecation warning.

#### DOM Events

The built-in components historically accepted a varierty of named arguments for
listening to certain DOM events on the component's HTML element. This includes
the following (may not be a complete list):

* `<LinkTo>`
  * `@change`
  * `@click`
  * `@contextMenu` (for the `contextmenu` event)
  * `@doubleClick` (for the `dblclick` event)
  * `@drag`
  * `@dragEnd` (for the `dragend` event)
  * `@dragEnter` (for the `dragenter` event)
  * `@dragLeave` (for the `dragleave` event)
  * `@dragOver` (for the `dragover` event)
  * `@dragStart` (for the `dragstart` event)
  * `@drop`
  * `@focusIn` (for the `focusin` event)
  * `@focusOut` (for the `focusout` event)
  * `@input`
  * `@keyDown` (for the `keydown` event)
  * `@keyPress` (for the `keypress` event)
  * `@keyUp` (for the `keyup` event)
  * `@mouseDown` (for the `mousedown` event)
  * `@mouseEnter` (deprecated, for the `mouseenter` event)
  * `@mouseLeave` (deprecated, for the `mouseleave` event)
  * `@mouseMove` (deprecated, for the `mousemove` event)
  * `@mouseUp` (for the `mouseup` event)
  * `@submit`
  * `@touchCancel` (for the `touchcancel` event)
  * `@touchEnd` (for the `touchend` event)
  * `@touchMove` (for the `touchmove` event)
  * `@touchStart` (for the `touchstart` event)
* `<Input>`
  * `@click`
  * `@contextMenu` (for the `contextmenu` event)
  * `@doubleClick` (for the `dblclick` event)
  * `@drag`
  * `@dragEnd` (for the `dragend` event)
  * `@dragEnter` (for the `dragenter` event)
  * `@dragLeave` (for the `dragleave` event)
  * `@dragOver` (for the `dragover` event)
  * `@dragStart` (for the `dragstart` event)
  * `@drop`
  * `@input`
  * `@mouseDown` (for the `mousedown` event)
  * `@mouseEnter` (deprecated, for the `mouseenter` event)
  * `@mouseLeave` (deprecated, for the `mouseleave` event)
  * `@mouseMove` (deprecated, for the `mousemove` event)
  * `@mouseUp` (for the `mouseup` event)
  * `@submit`
  * `@touchCancel` (for the `touchcancel` event)
  * `@touchEnd` (for the `touchend` event)
  * `@touchMove` (for the `touchmove` event)
  * `@touchStart` (for the `touchstart` event)
  * `@focus-in` (for the `focusin` event)
  * `@focus-out` (for the `focusout` event)
  * `@key-down` (for the `keydown` event)
  * `@key-press` (for the `keypress` event)
  * `@key-up` (for the `keyup` event)
* `<Textarea>`
  * `@click`
  * `@contextMenu` (for the `contextmenu` event)
  * `@doubleClick` (for the `dblclick` event)
  * `@drag`
  * `@dragEnd` (for the `dragend` event)
  * `@dragEnter` (for the `dragenter` event)
  * `@dragLeave` (for the `dragleave` event)
  * `@dragOver` (for the `dragover` event)
  * `@dragStart` (for the `dragstart` event)
  * `@drop`
  * `@input`
  * `@mouseDown` (for the `mousedown` event)
  * `@mouseEnter` (deprecated, for the `mouseenter` event)
  * `@mouseLeave` (deprecated, for the `mouseleave` event)
  * `@mouseMove` (deprecated, for the `mousemove` event)
  * `@mouseUp` (for the `mouseup` event)
  * `@submit`
  * `@touchCancel` (for the `touchcancel` event)
  * `@touchEnd` (for the `touchend` event)
  * `@touchMove` (for the `touchmove` event)
  * `@touchStart` (for the `touchstart` event)
  * `@focus-in` (for the `focusin` event)
  * `@focus-out` (for the `focusout` event)
  * `@key-down` (for the `keydown` event)
  * `@key-press` (for the `keypress` event)
  * `@key-up` (for the `keyup` event)

These arguments are no longer necessary – with angle bracket invocations, DOM
event listeners can be registered directly using the `{{on}}` modifier. An
invocation passing one or more of these named arguments shall trigger a
deprecation warning simialr to this:

```
<Input @click={{this.onClick}} />
       ~~~~~~~~~~~~~~~~~~~~~~~
or

{{input click=this.onClick}}
        ~~~~~~~~~~~~~~~~~~

Passing the `@click` argument to <Input> is deprecated. Instead, please use the
{{on}} modifier, i.e. `<Input {{on "click" ...}} />` instead of
`<Input @click={{...}} />` or `{{input click=...}}`.
```

Note that these named arguments were not necessarily an intentional part of the
component's original design. Rather, these are callbacks that would have fired
on all classic components, and since classic components' arguments are set on
the component instances as properties, passing these arguments at invocation
time would have "clobbered" any callbacks with the same name defined on the
component's class/prototype, whether it was intended by the component's author
or not.

For instance, the `<Input>` and `<Textarea>` built-in components implemented
callbacks that would have been clobbered by these named arguments (may not be a
complete list):

* `@change`
* `@focusIn`
* `@focusOut`
* `@keyDown`
* `@keyPress`
* `@keyUp`

Passing these named arguments historically supressed certain behavior of the
built-in components, in some cases preventing the components from functioning
properly. This was never an intended part of the original design and should be
considered a bug.

This bug may be fixed at any time during the transition period – the supression
behavior may stop without notice and should not be relied upon. An invocation
with these named arguemnts shall trigger a deprecation warning with this
additional caveat, similar to this:

```
<Input @change={{this.onChange}} />
       ~~~~~~~~~~~~~~~~~~~~~~~~~
or

{{input change=this.onChange}}
        ~~~~~~~~~~~~~~~~~~~~

Passing the `@change` argument to <Input> is deprecated. This would have
overwritten the internal `change` method on the <Input> component and prevented
it from functioning properly. Instead, please use the {{on}} modifier, i.e.
`<Input {{on "change" ...}} />` instead of `<Input @change={{...}} />` or
`{{input change=...}}`.
```

### No Longer Recommended

#### Changing `@tagName` on `<LinkTo>`

Due to the classic component implementation heritage, the built-in components
historically accepted a `@tagName` argument that allows customizing the tag
name of the underlying HTML element.

This was once popular with the `<LinkTo>` component for adding navigation
behavior to buttons, table row and other UI elements. The current concensus is
that this is an anti-pattern and causes issues with assistive technologies.

In most cases, the `<a>` anchor HTML element should be used for navigational UI
elements and styled with CSS to fit with the design requirements. Ocasionally,
a button may be acceptable, in which case a custom event handler can be written
using the router service and attached using the `{{on}}` modifier.

Other edge cases exists, but generally those solutions can be adapted to fufill
the requirements. For example, to make a table row clickable as a convenience,
the primary column can be made into a link, while a click event handler is
attached to the table row to redispatch the click to trigger the link.

Since this feature is no longer recommended, invoking `<LinkTo>` with the
`@tagName` argument shall trigger a deprecation warning similar to this:

```
<LinkTo @tagName="div" ...>...</LinkTo>
        ~~~~~~~~~~~~~~
or

{{#link-to tagName="div" ...}}...{{/link-to}}
           ~~~~~~~~~~~~~

Passing the `@tagName` argument to <LinkTo> is deprecated. Using a <div>
element for navigation is not recommended as it creates issues with assistive
technologies. Remove this argument to use the default <a> element. In the rare
cases that calls for using a different element, refactor to use the router
service inside a custom event handler instead.
```

Note that while the `<Input>` and `<Textarea>` element also accepted the
`@tagName` argument, it was never supported and its behavior is undefined. This
may stop "working" at any point without warning and should not be relied upon.

### Other Unsupported Arguments

Other named arguments not explicitly mentioned above are considered private
implementation details. Due to the nature of classic components' arguments
being set on its instance, any internal properties and methods could have been
clobbered by a named argument with the same name.

Some examples include private properties like `@active` and `@eventName` on
`<LinkTo>`, `@bubbles` and `@cancel` on `<Input>` and `<Textarea>`, lifecycle
hooks inherited from the classic component super class like `@didRender`,
`@willDestroy` and so on.

Clobbering these intenral properties and methods cause the components to behave
in unexpected ways. This should be considered a bug and should not be relied
upon. Any accidental difference in behavior caused by passing these unsupported
named arguments may stop at any time without warning.

## How we teach this

The implementation plan primarily relies on good deprecation messages to inform
users of the deprecated features and their migration paths. Deprecation guides
should be written based on the content and suggestions in this RFC. API docs
should be updated to mark these arguments as deprecated.

Once this migration is complete, the built-in components will be much easier to
teach, as they will have a small, well-defined API surface.

## Drawbacks

None.

## Alternatives

Instead of enumerating the supported arguments, we could attempt to enumerate
all the unsupported ones. However, this is quite difficult as any internal or
inherited properties could in theory be used as an invocation argument.

## Unresolved questions

None.
