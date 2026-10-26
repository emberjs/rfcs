---
stage: accepted
start-date: 2026-08-14T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
  - learning
  - typescript
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1225'
project-link:
---

# Deprecate the `<Input>` and `<Textarea>` components

## Summary

Deprecate the built-in `<Input>` and `<Textarea>` components, their curly forms
(`{{input}}`, `{{textarea}}`), and their `Input` / `Textarea` exports from
`@ember/component`. Use the native `<input>` and `<textarea>` elements with `{{on}}`
instead.

## Motivation

These components date from a time when templates could not bind attributes to elements
and two-way binding was how Ember worked. Neither is true anymore.
`<input value={{this.name}}>` has worked since HTMLBars, and Glimmer components removed
two-way binding on purpose. `<Input>` still installs one on `@value` and `@checked`:
when the user types, it assigns the new value to whatever you passed in.

The APIs these components were designed to pair with, `{{action}}`
([RFC #1006](./1006-deprecate-action-template-helper.md)) and `TargetActionSupport`
([RFC #1041](./1041-deprecate-target-action-support.md)), are already deprecated.

[RFC #671](./0671-modernize-built-in-components-1.md) made the classes private and
[RFC #707](./0707-modernize-built-in-components-2.md) removed all but six arguments.

The components also misbehave in ways the native elements do not. In
[ember.js#19222](https://github.com/emberjs/ember.js/issues/19222),
`<Input @value={{this.name}} {{on "input" this.validate}} />` gives `validate` a value
that is one keystroke stale, because `{{on}}` runs before the component updates the
binding. `<Input @type="not-a-type" />` silently renders `type="text"` after a runtime
feature-detection check that ships to every app. `<Input @type="checkbox" @value={{x}} />`
exists only to throw an assertion pointing you at `@checked`.

### Prior discussion

[emberjs/rfcs#498](https://github.com/emberjs/rfcs/issues/498), "Deprecate builtin input
and textarea components", was opened by `@mehulkar` in 2019 to gather community feedback.
It was closed for inactivity in 2022 with a note that the idea was still valid and needed
someone to pick it up. Other threads are linked in the
[appendix](#appendix-community-discussion).

> They do not bring a lot of value and they are one more Ember thing. Should you use the
> built in `<Input />` or the `<input type="text" />`? And why there is a component for
> the `textarea` and `input`, but not for `select`?
>
> -- `@mupkoo`

`@rwjblue` objected that the replacement was worse, and `@locks` agreed:

> In general, what I've seen folks suggest is something like:
>
> ```hbs
> <input value={{this.myValue}} oninput={{action (mut this.myValue) value="target.value"}}>
> ```
>
> I'm not convinced that this is better than using:
>
> ```hbs
> <Input @value={{this.myValue}} />
> ```
>
> -- `@rwjblue`

That was fair in 2019. `{{action}}` has since been deprecated, so the comparison is now:

```hbs
<input value={{this.myValue}} {{on "input" this.updateValue}}>
<Input @value={{this.myValue}} />
```

The native version is still longer and needs an `updateValue` handler, but the write to
`myValue` is in the app's own code.

## Transition Path

Deprecate:

1. `<Input />`, `<Textarea />`, `{{input}}`, `{{textarea}}`,
   `{{component "input"}}`, `{{component "textarea"}}`, and owner lookups of
   `component:input` / `component:textarea`.
2. The `Input` and `Textarea` exports from `@ember/component`, plus their TypeScript
   signatures and `Ember.Templates.components` registry entries.

`<LinkTo>` is out of scope. It has no native equivalent.

### Deprecation ids and messages

| id                                   | for            | until   |
| ------------------------------------ | -------------- | ------- |
| `ember.built-in-components.input`    | `ember-source` | `8.0.0` |
| `ember.built-in-components.textarea` | `ember-source` | `8.0.0` |

The ids follow the earlier `ember.built-in-components.import` and
`ember.built-in-components.legacy-arguments` deprecations.

The message must include the template location:

```text
The built-in `<Input>` component is deprecated. Use a native `<input>` element with
the `{{on}}` modifier instead. ('my-app/components/search.gjs' @ L12:C4) See
https://deprecations.emberjs.com/id/ember-built-in-components-input
```

### Migration

Replace each component with its native element, bind `value` or `checked`, and update
state from an `{{on}}` handler.

Glimmer sets `value` and `checked` on `<input>` and `<textarea>` as properties, so
`value={{this.term}}` stays in sync the way `@value` did, minus the write-back.

The [deprecation guides](#deprecation-guides) below cover each case in full.

### Lint rules

`eslint-plugin-ember` already flags usage:

- `ember/no-builtin-form-components` flags `Input` and `Textarea` imports from
  `@ember/component`.
- `ember/template-no-builtin-form-components` flags `<Input>` and `<Textarea>` in
  templates.

Instead of a separate codemod, `ember/template-no-builtin-form-components` should gain
an autofix for the common cases: an existing setter, or a binding that is easy to replace
with an `{{on}}` handler. It should join the `recommended` config when the deprecation
ships.

These rules become obsolete when the components are removed:

- `ember/template-no-input-block`
- `ember/template-no-input-tagname`
- `ember/template-builtin-component-arguments`

`ember/template-no-unknown-arguments-for-builtin-components` stays, because it also
checks `<LinkTo>`. It drops its `Input` and `Textarea` entries.

### Other ecosystem effects

- Addons that render `<Input>` will trigger the deprecation in apps that consume them,
  and those apps cannot fix it themselves.
- Glint already types native elements, so migrating does not lose type coverage. Editors
  stop suggesting `<Input>` and `<Textarea>` once the Glint signatures are removed.
- No effect on Ember Inspector, FastBoot, Engines, blueprints, or Ember Data.

## How We Teach This

- Open a PR to [`ember-learn/deprecation-app`](https://github.com/ember-learn/deprecation-app)
  with the two [deprecation guides](#deprecation-guides) below.
- Cut `<Input>` and `<Textarea>` from the Built-in Components guide, leaving `<LinkTo>`.
- Add a Forms guide: native elements, `{{on}}`, reading `event.target.value`, controlled
  vs uncontrolled inputs, form submission, labels and accessibility, file inputs.
- Audit the tutorial and mark the API docs deprecated.

### Deprecation guides

Add these two files to `content/ember/v7/` in `ember-learn/deprecation-app`. Set `since`
to the release that ships the deprecation.

#### `ember-built-in-components-input.md`

```md
---
title: The built-in `<Input>` component
until: 8.0.0
since: 7.x.0
displayId: ember.built-in-components.input
---
```

The built-in `<Input>` component is deprecated in all its forms: `<Input>`, `{{input}}`,
`{{component "input"}}`, and the `Input` export from `@ember/component`. Use the native
`<input>` element with the `{{on}}` modifier instead.

`<Input>` writes the new value back to whatever you pass to `@value` or `@checked`. A
native `<input>` doesn't, so read the value from the event and set it yourself.

If the deprecation points at a template inside an addon, update that addon or report
the usage to its maintainers.

To find every usage in your app, enable these `eslint-plugin-ember` rules:

- `ember/no-builtin-form-components`
- `ember/template-no-builtin-form-components`

##### Text inputs

Before:

```gjs
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { Input } from '@ember/component';

export default class Search extends Component {
  @tracked term = '';

  <template>
    <Input @value={{this.term}} placeholder="Search" />
  </template>
}
```

After:

```gjs
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { on } from '@ember/modifier';

export default class Search extends Component {
  @tracked term = '';

  setTerm = (event) => (this.term = event.target.value);

  <template>
    <input value={{this.term}} placeholder="Search" {{on "input" this.setTerm}} />
  </template>
}
```

##### Checkboxes

Bind `checked` and read `event.target.checked` in the handler.

Before:

```hbs
<Input @type="checkbox" @checked={{this.isEmberized}} name="isEmberized" />
```

After:

```gjs
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { on } from '@ember/modifier';

export default class Preferences extends Component {
  @tracked isEmberized = false;

  toggleEmberized = (event) => (this.isEmberized = event.target.checked);

  <template>
    <input
      type="checkbox"
      checked={{this.isEmberized}}
      name="isEmberized"
      {{on "change" this.toggleEmberized}}
    />
  </template>
}
```

##### Template-only components

A template-only component has no class to hold a setter. Put the handler in module
scope and pass it the object to update. You can also convert the component to a class.

Before:

```gjs
import { Input } from '@ember/component';

<template>
  <Input @value={{@model.name}} />
</template>
```

After:

```gjs
import { fn } from '@ember/helper';
import { on } from '@ember/modifier';

const setName = (model, event) => (model.name = event.target.value);

<template>
  <input value={{@model.name}} {{on "input" (fn setName @model)}} />
</template>
```

##### `@enter` and `@insert-newline`

These usually mean "submit". Wrap the input in a `<form>` and handle `submit` instead;
the browser submits the form on Enter.

Add a `search` method to the `Search` class from the text input example. Call
`event.preventDefault()` so the page does not reload:

```gjs
search = (event) => {
  event.preventDefault();
  this.args.onSearch(this.term);
};
```

```gjs
<form {{on "submit" this.search}}>
  <label for="term">Search</label>
  <input id="term" name="term" value={{this.term}} {{on "input" this.setTerm}} />
  <button type="submit">Search</button>
</form>
```

##### `@escape-press` and other keys

Listen for `keydown` and check `event.key`:

```gjs
cancelOnEscape = (event) => {
  if (event.key === 'Escape') {
    this.cancel();
  }
};
```

```gjs
<input value={{this.term}} {{on "keydown" this.cancelOnEscape}} />
```

The `{{on-key "Escape" this.cancel}}` modifier from
[`ember-keyboard`](https://github.com/adopted-ember-addons/ember-keyboard) also works.

#### `ember-built-in-components-textarea.md`

```md
---
title: The built-in `<Textarea>` component
until: 8.0.0
since: 7.x.0
displayId: ember.built-in-components.textarea
---
```

The built-in `<Textarea>` component is deprecated in all its forms: `<Textarea>`,
`{{textarea}}`, `{{component "textarea"}}`, and the `Textarea` export from
`@ember/component`. Use the native `<textarea>` element with the `{{on}}` modifier
instead.

`<Textarea>` writes the new value back to whatever you pass to `@value`. A native
`<textarea>` doesn't, so read the value from the event and set it yourself.

If the deprecation points at a template inside an addon, update that addon or report
the usage to its maintainers.

To find every usage in your app, enable these `eslint-plugin-ember` rules:

- `ember/no-builtin-form-components`
- `ember/template-no-builtin-form-components`

Before:

```gjs
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { Textarea } from '@ember/component';

export default class Notes extends Component {
  @tracked notes = '';

  <template>
    <Textarea @value={{this.notes}} rows="4" />
  </template>
}
```

After:

```gjs
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { on } from '@ember/modifier';

export default class Notes extends Component {
  @tracked notes = '';

  setNotes = (event) => (this.notes = event.target.value);

  <template>
    <textarea rows="4" value={{this.notes}} {{on "input" this.setNotes}}></textarea>
  </template>
}
```

For `@insert-newline` and `@escape-press`, listen for `keydown` and check `event.key`.
See the [`<Input>` guide](/id/ember-built-in-components-input) for an example.

## Drawbacks

- Usage is widespread. These components appear in most apps that have forms, in many
  addons, and in a decade of tutorials and Stack Overflow answers that will not be updated.
- Every field needs a handler: a method on a class or a function in the module. There
  is still no blessed template-level way to say "set this to the event's value".
  `@pzuraq` raised this in #498 and it is still open: `(mut)` is not the happy path, there
  are no template closures, and there is no `(set ... (pick ...))`.

## Alternatives

- Do nothing. The components keep working and keep costing maintenance and bundle size.
- Ship a build-time transform that rewrites them to native elements. Authors would still
  write `@value` and get its implicit write-back.
- Move them to an addon. This only moves the maintenance out of `ember-source`.

### Prior art

No other major framework ships an input component. Svelte (`bind:value`), Vue
(`v-model`), and Angular (`[(ngModel)]`) add binding syntax to the native element. React
and Solid use the native element as is.

## Unresolved questions

- Should Ember add a template-level way to set a value from an event (see Drawbacks)?
  This RFC does not depend on the answer.

## Appendix: community discussion

### Issues and RFCs

- [emberjs/rfcs#498](https://github.com/emberjs/rfcs/issues/498): the 2019 proposal to
  deprecate these components, closed for inactivity in 2022.
- [`@stefanpenner` on Twitter](https://twitter.com/stefanpenner/status/1136332615923847168):
  the 2019 thread that prompted #498.
- [ember.js#19222](https://github.com/emberjs/ember.js/issues/19222): `{{on "input"}}`
  fires before `<Input>` updates its binding.
- [ember.js#18249](https://github.com/emberjs/ember.js/issues/18249): adding a `change`
  action breaks the two-way binding on `checked`.
- [ember.js#19763](https://github.com/emberjs/ember.js/issues/19763): a checkbox `change`
  handler behaves differently from 3.27 on.

### Forum threads

- [Ember Octane and 2-Way-Binding. What are you recommend?](https://discuss.emberjs.com/t/ember-octane-and-2-way-binding-what-are-you-recommend/18092)
  (2020): whether new Octane apps should use `<Input>` or a native input.
- [Correct way to pass arguments to built-in components with angle-bracket syntax](https://discuss.emberjs.com/t/correct-way-to-pass-arguments-to-built-in-components-with-angle-bracket-syntax/16635)
  (2019): confusion over which `<Input>` options are arguments and which are attributes.
- [Input Helper fires action before binding value](https://discuss.emberjs.com/t/input-helper-fires-action-before-binding-value/19310)
  (2021) and
  [Input component value lagging one change behind](https://discuss.emberjs.com/t/input-component-value-laging-one-change-behind/19633)
  (2022): users hitting ember.js#19222.

### Posts

- Jen Weber, ["Building an Octane-style input"](https://www.jenweber.dev/building-an-octane-style-input/):
  building a text input on the native element. It covers where `<Input>` falls short on
  data flow, validation, and accessibility, though the author still likes its two-way binding.
