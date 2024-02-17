---
stage: accepted
start-date: 2023-12-22T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - framework
  - learning
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/997
project-link:
suite: 
---

<!--- 
Directions for above: 

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
suite: Leave as is
-->

# Make `{{on}}` a built in modifier  

## Summary

Today, when using gjs/gts/`<template>`, in order to bind event listeners, folks _must import_ the `{{on}}` modifier.
Because event listening is so commonplace, this is a grating annoyance for developers.

This RFC proposes that `{{on}}` be built in to `glimmer-vm` and not require importing.

## Motivation

Given how common it is to use the `{{on}}` modifier:

```gjs
import { on } from '@ember/modifier';

<template>
    <button {{on 'click' @doSomething}}>
        click me
    </button>

    <form {{on 'submit' @localSubmit}}>
        <label
            {{on 'keydown' @a}}
            {{on 'keyup' @a}}
            {{on 'focus' @a}}
            {{on 'blur' @a}}
        >
        </label>

        <button>
            submit
        </button>
    </form>
</template>
```

It should be built in to the templating engine, Glimmer, so that folks don't need to import it.

There is precedence for this already as the following are already commonplace and built in:
- `(helper)`
- `(modifier)`
- `(component)`

It's historically been the stance that, 

"If it can be built in userspace, it should be, leaving the framework-y things to be only what can't exist in userspace"

But to achieve the ergonomics that our users want, we should provide a more cohesive experience, rather than require folks import from all of (in the same file):
- `@glimmer/component`
- `@glimmer/tracking`
- `@ember/modifier`
- `@ember/helper`
- `ember-modifier`
- `ember-resources`
- `tracked-built-ins`

Some of the above may unify in a separate RFC, but for the template-utilities, since the modules that provide them are already so small, it makes sense to inherently provide them by default. Especially as we can target strict-mode only, so that we don't run in to the same implementation struggles that built-in [Logical Operators](https://github.com/emberjs/rfcs/pull/562), [Numeric Comparison Operators](https://github.com/emberjs/rfcs/pull/561), and [Equality Operators](https://github.com/emberjs/rfcs/pull/560) have faced.

<details><summary>some context on those RFCs</summary>

The main problem with adding default utilities without strict-mode is that it becomes very hard to implement a way for an app to incrementally, and possibly per-addon, or per-file, to adopt the default thing due to how resolution works. Every usage of the built in utility would also require a global resolution lookup (the default behavior in loose mode templates) to see if an addon is overriding the built ins -- and then, how do you opt in to the built ins, and _not_ let addons override what you want to use?

With gjs/gts/`<template>`, this is much simpler, as in strict-mode, you can check if the scope object defines the helpers, and if not, use the built in ones.

This strategy of always allowing local scope to override default-provided utilities will be a recurring theme.

</details>

---------------

_Making `on` a built-in will help make writing components feel more cohesive and well supported, as folks will not need to cobble together many imported values_

----------------

## Detailed design

This change would affect strict-mode only. This is so that today's existing code that imports `on` from `@ember/modifier` will still work due to how values defined locally in scope override globals.

The behavior of `on` would be the same as it is today, but defined by default in the `glimmer-vm`.


`on` will be a keyword, and for backwards compatibility, this will require that keywords, in strict mode, be overrideable by the strict-mode scope bag.


## How we teach this

Once implemented, the guides, if they say anything about gjs/gts/`<template>` and `on` by the time this would be implemented, would only remove the import.

## Drawbacks

People may not know where `on` is defined.
- counterpoint: do they need to?, we are defining a lanugage, trying to make it ergonomic.

We need to allow keywords to be overridable in Glimmer -- this is a behavior most languages do not allow.

## Alternatives

- Use a prelude
    - preludes were mentioned during the initial exploration of strict-mode templates, and were decided against, because addons would not be able to assume a prelude exists, as apps could define their own, and this sort of re-introduces the app-tree-merging behavior that we've been trying to get away from. 

- Use an alternate syntax: `on:click={{handler}}` or `on:{eventname}={{value}}`
    - This would be even more ergonomic, and I think we should do this syntax anyway, but may take longer to implement. -- thought would not require glimmer allow the scope bag to overrid keywords.
        

## Unresolved questions

- What happens if we want to remove a keyword? (like `mut`)
  - same as today, we only need to commit to a major to remove the keyword in and then do it - providing ample deprecation time, ending with the final removal.
