- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Return noop function if {{fn}} helper is invoked without arguments

## Summary

This RFC introduces an argumentless version of the `{{fn}}` template helper.
It should return a noop function.

The RFC does not intend to change the existing meaning of the `{{fn}}` template
helper if invoked with one or more arguments. Especially it does *not* propose
to support `undefined` as value for the first positional argument. The `{{fn}}`
template helper should continue to throw an assertion if invoked with another
value than a function as the first positional argument.

## Motivation

This RFC is driven by two motivations:

1. Provide a built-in way to get a noop function in a template.
2. Simplify mental model around `{{fn}}` helper.

### Use cases for noop functions in templates

Noop function are commonly used in modern Ember. Currenlty the framework misses
a built-in way to get one in a template. Developers need to either reach out
for a userland tempalte helper like the [`{{noop}}` helper from
ember-composable-helpers](https://github.com/DockYard/ember-composable-helpers#noop)
package or expose a noop function from backing JavaScript class.

A noop function is mainly required in components that have an optional
argument, which should be either passed down to another component as an action
handler or used with an event listener registered through `{{on}}` modifier.
The value of the optional argument is `undefined` if consumer does not set it
but the component may and the `{{on}}` modifier do not support `undefined` but
require a function.

Let's have a look at two example use cases to illustrate the issue:

1. Let's consider a `<FancyModal>` component, which should have an optional
   `onHidden` argument. The `onHidden` argument accepts a function, which is
   invoked when the modal is not visible anymore.

   The `<FancyModal>` component uses a `<BasicModal>` component internally.
   That `<BasicModal>` component also has a `onHidden` argument. It's optional
   but if set it must be a function. Especially `undefined` is not a supported
   value.

   Depending if a consumer of `<FancyModal>` uses `onHidden` argument,
   `@onHidden` value in `<FancyModal>`'s template may be either a function or
   `undefined`. As `<BasicModal>` does not support `undefined` as a value for
   its `onHidden` argument, the developer must conditionally either pass
   `@onHidden` or a noop function to `<BasicModal>`'s `onHidden` argument.

   An implementation of `<FancyModal>` as a template-only component using
   `{{noop}}` template helper from ember-composable-helpers would look like
   this:

   ```hbs
   <BasicModal
     @onHidden={{if @onHidden @onHidden (noop)}}
   />
   ```

   This RFC would allow the project to drop the dependency on
   ember-composable-helpers and refactor the component like this:

   ```patch
     <BasicModal
   -   @onHidden={{if @onHidden @onHidden (noop)}}
   +   @onHidden={{if @onHidden @onHidden (fn)}}
     />
   ```

2. Let's consider a `<Button>` component, which should have an optional
   argument `onClick`. It should accept a function, which is invoked if the
   user clicks the button.

   The `{{on}}` modifier requires the its first positional argument to be a
   function. An assertion is thrown if `undefined` is passed. As in the first
   example the developer need to check if `@onClick` argument is set and pass
   a noop function otherwise.

   An implementation of the `<Button>` component as a template-only component
   using `{{noop}}` template helper from ember-composable-helpers would look
   like this:

   ```hbs
   <button {{on "click" (if @onClick @onClick (noop))}}>
     {{yield}}
   </button>
   ```

   This RFC would allow the project to drop the dependency on
   ember-composable-helpers and refactor the component like this:

   ```patch
   - <button {{on "click" (if @onClick @onClick (noop))}}>
   + <button {{on "click" (if @onClick @onClick (fn))}}>
       {{yield}}
     </button>
   ```

> Both examples could be simplified if
> [RFC 562: Adding Logical Operators to Templates](https://github.com/cibernox/rfcs/blob/add-logical-operators-to-templates/text/0000-add-logical-operators.md)
> lands:
>
> ```patch
>   <BasicModal
> -   @onHidden={{if @onHidden @onHidden (fn)}}
> +   @onHidden={{or @onHidden (fn)}}
>   />
> ```
>
> ```patch
> - <button {{on "click" (if @onClick @onClick (fn))}}>
> + <button {{on "click" (or @onClick (fn))}}>
>      {{yield}}
>   </button>
> ```

A noop function will not be needed for `{{on}}` modifier anymore after
[RFC 432: Contextual Helpers and Modifiers (a.k.a. "first-class helpers/modifiers")](https://emberjs.github.io/rfcs/0432-contextual-helpers.html)
is implemented. This RFC extends the Glimmer template syntax to conditionally
apply modifiers (and helpers). The second example should be refactored after
contextual modifiers are available to only invoke the `{{on}}` modifier if
`@onClick` is passed:

```patch
- <button {{on "click" (if @onClick @onClick (fn))}}>
+ <button {{if @onClick (modifier on "click" @onClick)}}>
    {{yield}}
  </button>
```

There isn't a similar example for the first example as the Glimmer template
syntax do not support conditionally setting arguments on component invocation.

### Simplification of the mental model around `{{fn}}` template helper

The mental model around `{{fn}}` template helper is currenlty focues on partial
application of arguments to an existing function. As this is a rather complex
technical concept it introduces a high entry barrier for developers who are not
familiar with that concept.

This RFC would lower that entry barrier. It would allow reasoning about the
`{{fn}}` template helper as a helper which returns a function. Teaching it
would become a process of small incremental steps, which do not require
knowledge of the partial application pattern upfront but rather would teach
it:

1. The `{{fn}}` template helper returns a function.
2. If invoked without any argument, it's a noop function, which does nothing.
3. It accepts another function as an optional first argument. If set, the
   function returned will invoke that function. All arguments received are
   passed through.
4. You may pass additional positional arguments, which are set on the function
   passed as first argument before the once passed through.

This is discussed in *How we teach this* section more in detail.

### Historical context

It may be confusing why this capability was not introduced for `{{fn}}`
template helper from the beginning. To understand that one we need to have
a look on the evolution of [RFC 470: `{{fn}}` helper](https://emberjs.github.io/rfcs/0470-fn-helper.html)
which introduced it.

The RFC was focused on replacing the `{{action}}` template helper. An
argumentless version of the helper introduced or using it to get a noop
function was not discussed at all.

This could be best understood from the evolution of the RFC. It evolved in
three phases:

1. In the beginning it tried to solve both: context binding and partial
   application. Actually it was more focused on context binding than on partial
   application of argument. This was reflected in the proposed name `{{bind}}`.
2. After a discussion in core team the RFC changed to focus on partial
   application only. Context binding was solved by `@action` decorator instead
   and fully removed from the RFC 470. The proposed name for that template
   helper was changed to `with-args`.
3. Nearly at the last minute the name was changed to `{{fn}}`. The RFC was moved
   to final comment period at the same time.

## Detailed design

The implementation of `{{fn}}` template helper should be changed to return a
noop function if its invoked without any argument.

The existing `{{fn}}` helper could be implemented like this in userland:

```js
import { helper } from '@ember/component/helper';
import { assert } from '@ember/debug';

export default helper(function fn([fn, ...arguments]) {
  assert(
    `You must pass a function as the \`fn\` helpers first argument.`,
    typeof fn === 'function'
  );

  return fn.call(this, ...arguments);
});
```

> The [actual implementation](https://github.com/emberjs/ember.js/blob/v3.22.0/packages/@ember/-internals/glimmer/lib/helpers/fn.ts)
> is way different from that pseudo code as it's implemented in the Glimmer VM.

To implement this RFC that pseudo code would change like this:

```patch
  import { helper } from '@ember/component/helper';
  import { assert } from '@ember/debug';

+ const NOOP_FUNCTION = function() {};
+
- export default helper(function fn([fn, ...arguments]) {
+ export default helper(function fn(params) {
+   if (params.length === 0) {
+     return NOOP_FUNCTION;
+   }
+
+   const [fn, ...arguments] = params;

    assert(
      `You must pass a function as the \`fn\` helpers first argument.`,
      typeof fn === 'function'
    );

    return fn.call(this, ...arguments);
  });
```

Beside adding an argumentless version of the `{{fn}}` helper, this RFC also
clarifies that invoking the helper with exactly one argument is supported
as long as it's a function. This is not explicitly stated in RFC 470 but
matches the existing implementation.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

1. The RFC slightly increases public API surface.
2. Developer may use a noop function instead of conditionally invoking a
   modifier even after *RFC 432: Contextual Helpers and Modifiers (a.k.a.
   "first-class helpers/modifiers")* has landed.

## Alternatives

1. `{{fn}}` helper could support `undefined` as a potential value for it's
   first positional argument and return a noop function instead of throwing
   an assertion. This would have the major trade-off that it's usage would be
   less explicit and that it would be harder to debug if a typo causes a value
   to be `undefined` rather than the expected function.

2. Glimmer template syntax could be extended to allow conditional setting an
   argument on component invocation similar to how contextual modifiers allow
   conditional invoking a modifier. This would reduce the long-term need for
   noop functions dramatically.

## Unresolved questions

No unresolved question has been discovered so far.
