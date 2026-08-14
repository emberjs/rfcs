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
and two-way binding was how Ember worked. Neither is true anymore:

- `<input value={{this.name}}>` has worked since HTMLBars.
- Glimmer components removed two-way binding on purpose, but `<Input>` still installs one
  on `@value` and `@checked`.
- `{{action}}` ([RFC #1006](./1006-deprecate-action-template-helper.md)) and
  `TargetActionSupport` ([RFC #1041](./1041-deprecate-target-action-support.md)), the
  APIs these components were designed to pair with, are already deprecated.
- In `.gjs` you have to write `import { Input } from '@ember/component'`. The native
  element needs no import, so the built-in is now the longer option.

[RFC #671](./0671-modernize-built-in-components-1.md) made the classes private and
[RFC #707](./0707-modernize-built-in-components-2.md) removed all but six arguments. What
is left is a two-way binding the framework tells you not to use, plus three invented
events (`@enter`, `@insert-newline`, `@escape-press`).

They also misbehave in ways the native elements do not. In
[ember.js#19222](https://github.com/emberjs/ember.js/issues/19222),
`<Input @value={{this.name}} {{on "input" this.validate}} />` gives `validate` a value
that is one keystroke stale, because `{{on}}` runs before the component updates the
binding. `<Input @type="fooo" />` silently renders `type="text"` after a runtime
feature-detection check that ships to every app. `<Input @type="checkbox" @value={{x}} />`
exists only to throw an assertion pointing you at `@checked`.

### Prior discussion

[emberjs/rfcs#498](https://github.com/emberjs/rfcs/issues/498), "Deprecate builtin input
and textarea components", was opened by `@mehulkar` in 2019 and closed for inactivity in
2022 with a note that the idea was still valid and needed someone to pick it up. The
arguments from that thread still apply:

> It seems like an arbitrary set of helpers. There are other interactive elements that are
> in the same problem space, but ember doesn't provide anything for that.
> -- `@mehulkar`

> 1. Glimmer components now force people to use DDAU and get rid of two way binding in
>    general. 2. At the same time ember provides components to exactly do _not_ that.
> -- `@gossi`

> They do not bring a lot of value and they are one more Ember thing. Should you use the
> built in `<Input />` or the `<input type="text" />`? And why there is a component for
> the `textarea` and `input`, but not for `select`?
> -- `@mupkoo`

> I personally found trying to reimplement the DOM behavior in userland an endless source
> of complexity bugs and confusion, with little benefit. ember checkbox/select/input are
> examples of that.
> -- `@stefanpenner`, [quoted in the thread](https://twitter.com/stefanpenner/status/1136332615923847168)

The objection at the time, from `@rwjblue` and `@locks`, was that the replacement was
worse:

> ```hbs
> <input value={{this.myValue}} oninput={{action (mut this.myValue) value="target.value"}}>
> ```
> I'm not convinced that this is better than using `<Input @value={{this.myValue}} />`.

That was fair in 2019. Both `{{action}}` and `(mut)` are deprecated now, `{{on}}` is the
answer, and in `.gjs` the handler lives next to the template.

## Transition Path

Deprecate:

1. `<Input />`, `<Textarea />`, `{{input}}`, `{{textarea}}`,
   `{{component "input"}}`, `{{component "textarea"}}`, and owner lookups of
   `component:input` / `component:textarea`.
2. The `Input` and `Textarea` exports from `@ember/component`, plus their TypeScript
   signatures and `Ember.Templates.components` registry entries.
3. All remaining arguments: `@type`, `@value`, `@checked`, `@enter`, `@insert-newline`,
   `@escape-press`.

`<LinkTo>` is out of scope. It has no native equivalent.

Proposed ids `built-in-components.input` and `built-in-components.textarea`, `for:
'ember-source'`, `until: '8.0.0'`. The deprecation must report the template location so
app authors can tell their own usage apart from an addon's.

### Text inputs

```gjs
// before
import { Input } from '@ember/component';

export default class Search extends Component {
  @tracked term = '';

  <template>
    <Input @value={{this.term}} placeholder="Search" />
  </template>
}
```

```gjs
// after
import { on } from '@ember/modifier';

export default class Search extends Component {
  @tracked term = '';

  setTerm = (event) => (this.term = event.target.value);

  <template>
    <input value={{this.term}} placeholder="Search" {{on "input" this.setTerm}} />
  </template>
}
```

The handler reads the value off the event, so it cannot go stale the way #19222 does.

### Checkboxes

```hbs
{{! before }}
<Input @type="checkbox" @checked={{this.isEmberized}} name="isEmberized" />

{{! after }}
<input
  type="checkbox"
  checked={{this.isEmberized}}
  name="isEmberized"
  {{on "change" this.toggleEmberized}}
/>
```

### Textareas

```hbs
{{! before }}
<Textarea @value={{this.notes}} rows="4" />

{{! after }}
<textarea rows="4" value={{this.notes}} {{on "input" this.setNotes}}></textarea>
```

Glimmer treats `value` on `<input>` and `<textarea>` as a property, so this binds the
live value like `@value` did, without the write-back.

### `@enter`, `@insert-newline`, `@escape-press`

`@enter` on a text field is usually form submission. A real form handles it, and gets
native validation and correct behavior for assistive tech:

```gjs
<form {{on "submit" this.search}}>
  <label for="term">Search</label>
  <input id="term" name="term" value={{this.term}} {{on "input" this.setTerm}} />
  <button type="submit">Search</button>
</form>
```

Otherwise, use `keydown` and check `event.key`. This is the kind of thing a small
published modifier (`{{on-key "Escape" this.cancel}}`) handles well, and several addons
already ship one.

### Lint rules

`eslint-plugin-ember` should flag the `@ember/component` imports.

### Other ecosystem effects

Addons that render `<Input>` will trigger the deprecation in apps that consume them, and
those apps cannot fix it themselves. Glint already types native elements, so most apps
gain type coverage. No impact on FastBoot, Engines, blueprints, or Ember Data. Ember
Inspector's component tree gets smaller.

## How We Teach This

Ember should teach HTML here. `<input {{on "input" ...}}>` transfers to any other
codebase; `<Input @value={{...}} />` only works in Ember and contradicts the data flow the
guides teach elsewhere.

- Cut `<Input>` and `<Textarea>` from the Built-in Components guide, leaving `<LinkTo>`.
  The current text describes the two-way binding as a feature with no caveats.
- Add a Forms guide: native elements, `{{on}}`, reading `event.target.value`, controlled
  vs uncontrolled inputs, form submission, labels and accessibility, file inputs. The
  guides are missing this today because `<Input>` covered for it.
- Audit the tutorial, mark the API docs deprecated, and write a deprecation guide with
  the before/after examples above.

Jen Weber's ["Building an Octane-style input"](https://www.jenweber.dev/building-an-octane-style-input/)
and the recurring [forum threads](https://discuss.emberjs.com/t/ember-octane-and-2-way-binding-what-are-you-recommend/18092)
asking whether to use `<Input>` or a native input suggest the teaching materials have
been working around these components for years.

## Drawbacks

- Usage is large. These appear in most apps that have forms, in many addons, and in a
  decade of tutorials and Stack Overflow answers that will not be updated.
- The replacement is more code. Two-way binding is fewer characters, even though it is
  harder to debug.
- There is still no blessed template-level way to say "set this to the event's value".
  `@pzuraq` raised this in #498 and it is still open: `(mut)` is not the happy path, there
  are no template closures, and there is no `(set ... (pick ...))`. Every migration adds a
  method to a class.
- Apps will see deprecations from addon code they cannot fix themselves, as with any
  deprecation that reaches libraries.

## Alternatives

- Do nothing. The components keep working and keep costing maintenance, bundle size, and
  the "why is there an `<Input>` but no `<Select>`?" question.
- Ship a build-time transform that rewrites `<Input>` and `<Textarea>` to native elements,
  so apps get the output without touching their own code.
- Move them to an addon and keep recommending them. Fixes the maintenance cost, not the
  teaching problem.

No other major framework ships an input component. React, Svelte, Vue, and Solid all use
the native element, and where they add sugar it is a binding attached to that element
(`bind:value`, `v-model`, `[(ngModel)]`), not a replacement element. That is the shape of
the gap here.

## Unresolved questions

n/a
