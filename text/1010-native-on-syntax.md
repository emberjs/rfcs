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

# Better on / event-listening syntax

## Summary

Today, when using gjs/gts/`<template>`, in order to bind event listeners, folks _must import_ the `{{on}}` modifier.
Because event listening is so commonplace, this is a grating annoyance for developers.

This RFC proposes that our templating language should support event binding natively via a new syntax in element-space, similar to modifiers.

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
Additionally, since `on` is an alias for `addEventListener` and `removeEventListener`, making it stand out more compared to other kinds of modifiers will improve scannability of code, and allow us to optimize the cleanup of elements more aggressively.

---------------

_Making `on` have its own syntax will help make writing components feel more cohesive and well supported, as folks will not need to cobble together many imported values_

----------------

## Detailed design

<table>
<thead><tr><th>Current Syntax</th><th>Proposed Synatx</th></tr></thead>
<tr><td>

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

</td>
<td>



```gjs
<template>
    <button on:click={{@doSomething}}>
        click me
    </button>

    <form on:submit={{@localSubmit}}>
        <label
            on:keydown={{@a}}
            on:keyup={{@a}}
            on:focus={{@a}}
            on:blur={{@a}}
        >
        </label>

        <button>
            submit
        </button>
    </form>
</template>
```

</td></tr>></table>


The syntax can be interpreted as:

```hbs
<div on:eventName={{value}}></div>
```
where the Vanilla JavaScript would like:
```js
div.addEventListener(eventName, value);
```

Noting that: 
- Cleanup _may_ not need to happen if the element is removed, as event listeners on a removed element are cleaned up by the browser
- These styles of event listeners are never conditional -- if folks want conditional modifiers, they'll want to use `import { on } from '@ember/modifier';`


This new syntax must also introduce the concept of _Event Modifiers_, which are partly the configuration options as the [third argument to `addEventListener`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener).

These would be
- `capture`
- `once`
- `passive`

However, for convinience, and also solving a minor annoyance folks have with event handling, we'll want additional _Event Modifiers_[^event-modifier-inspo] as well.

For example, when working with `<form>`, and wanting to handle the submit client-side, folks must have a `event.preventDefault()` to prevent the default behavior of trying to submit the form-data to the server.

<table>
<thead><tr><th>Current Syntax</th><th>Proposed Synatx</th></tr></thead>
<tr><td>

```gjs
import Component from '@glimmer/component';
import { on } from '@ember/modifier';

export default class Demo extends Component {
    <template>
        <form {{on 'submit' this.handleSubmit}}>
            {{! ... }}

            <button>
                submit
            </button>
        </form>
    </template>

    handleSubmit = (event) => {
        event.preventDefault();

        this.args.onSubmit(...);
    }
}
```

</td>
<td>



```gjs
import Component from '@glimmer/component';
import { on } from '@ember/modifier';

export default class Demo extends Component {
    <template>
        <form on:submit|preventDefault={{@onSubmit}}>
            {{! ... }}

            <button>
                submit
            </button>
        </form>
    </template>
}
```

</td></tr>></table>

The full list of Event Modifiers:

- `preventDefault` — calls `event.preventDefault()` before running the handler. 
- `stopPropagation` — calls `event.stopPropagation()`, preventing the event reaching the next element
- `passive` — improves scrolling performance on touch/wheel events (Svelte will add it automatically where it's safe to do so)
- `capture` — fires the handler during the capture phase instead of the bubbling phase
- `once` — remove the handler after the first time it runs
- `self` — only trigger handler if event.target is the element itself
- `trusted` — only trigger handler if `event.isTrusted` is `true`, meaning the event was triggered by a user action rather than because some JavaScript called `element.dispatchEvent(...)`


Event modifiers could also be changed together:

```hbs
<button on:click|once|capture={{@hadleClick}}>
  ...
</button>
```

[^event-modifier-inspo]: [Borrowed from Svelte](https://learn.svelte.dev/tutorial/event-modifiers)

------------

This change would affect strict-mode only, but could be added to loose mode (whichever is easier) -- there may be downsides to increasing usage of the next syntax across both strict and loose mode.

Code that imports `on` from `@ember/modifier` will still work.


## How we teach this

Once implemented, the guides, if they say anything about gjs/gts/`<template>` and `on` by the time this would be implemented, would only remove the import.

## Drawbacks

- We may eventually want to deprecate the `on` modifier as it is used prior to this RFC for consistency reasons -- nothing more as this RFC does not propose any changes to existing `on` behavior.

## Alternatives

Unlike in [RFC 997](https://github.com/emberjs/rfcs/pull/997), we could make the `on` modifier available as a keyword in modifier-space. _however_ to make that work with maximum compatibility and not break existing code-bases, we would have to allow keywords to be overwritten by local scope.

## Unresolved questions

n/a
