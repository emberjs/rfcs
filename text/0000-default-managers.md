- Start Date: 2021-05-17
- Relevant Team(s): Ember.js, Learning
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Default Managers

## Summary

Anything that can be in a template has its lifecycle managed by the manager pattern.
Today, we have known managers for `@glimmer/component`, `@ember/helper`, etc.
But what happens when the VM encounters an object for which there is no manager?,
such as a plain function? This RFC explores and proposes a default behavior for
those unknown scenarios.

## Motivation

The addon, [ember-could-get-used-to-this](https://github.com/pzuraq/ember-could-get-used-to-this)
demonstrated that it's possible to use plain functions for helpers and modifiers.
And since Ember 3.25, helpers can be invoked directly from value references, this opened
a whole new world of ergonomics improvements where a dev could define a function in a
component class and use that function **as** a helper, thanks to
ember-could-get-used-to-this implementing a Helper Manager that _knew what to do with plain functions.

This has the impact of greatly reducing helper burden for apps and addon devs, in that,
folks no longer need to jump over to the app/addon helpers directory to create a helper
"just to do this one simple thing". It's all now `{{this.myHelper}}` or `{{this.myModifier}}`.

This has the added benefit of, with template strict mode, using plain functions in module-scope
for helpers and modifiers.

For Example:

<!--
  If you're reading this RFC non-rendered, ignore the "jsx" language tag.
  "jsx" is incorrect, and isn't perfect, but "gjs" syntax highlighting
  hasn't been added to GitHub's highlighting yet.
-->

```jsx
const double = num => num * 2;
const resizable = element => { /* ... */ };

<template>
  {{double 2}}
  <div {{resizable}}></div>
</template>
```
_Note: `<template>` is experimental syntax, and should not be be the focus of this example_

Another aspect of which is exciting is that it becomes easier, in tests to grab
the output of yielding components:

```jsx
test('...', async function (assert) {
  let output = [];
  const capture = (...args) => output = args;

  await render(hbs`
    <MyComponent as |yielded info and things|>
       {{capture yielded info and things}}
    </MyComponent>
  `);

  assert.equal(output[0], /* value of yielded */ );
  assert.equal(output[1], /* value of info */ );
  // ...
})
```

## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

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

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
