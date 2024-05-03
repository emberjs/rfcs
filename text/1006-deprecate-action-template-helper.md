---
stage: ready-for-release
start-date: 2024-02-13T00:00:00.000Z
release-date:
release-versions:
teams:
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1006'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/1011'
project-link:
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
-->

# Deprecate `(action)` template helper and `{{action}}` modifier. 

## Summary

The `(action)` template helper and `{{action}}` modifier was common pre-Octane. Now that we have native classes and the `{{on}}` modifier, we no longer need to use `(action)` or `{{action}}`

## Motivation

Remove legacy code with confusing semantics.

This is a part of _[Deprecating Ember Classic (pre-Octane)](https://github.com/emberjs/rfcs/issues/832)_.

## Transition Path

This was written in the [Octave vs Classic cheatsheet](https://ember-learn.github.io/ember-octane-vs-classic-cheat-sheet/#component-properties__ddau)

<details><summary>that content here</summary>

### Before (pre-Octane)

```js
// parent-component.js
import Component from '@ember/component';

export default Component.extend({
  count: 0
});

```
```hbs
{{!-- parent-component.hbs --}}
{{child-component count=count}}
Count: {{this.count}}

```
```js
// child-component.js
import Component from '@ember/component';

export default Component.extend({
  actions: {
    plusOne() {
      this.set('count', this.get('count') + 1);
    }
  }
});
```
```hbs
{{!-- child-component.hbs --}}
<button type="button" {{action "plusOne"}}>
  Click Me
</button>
```

### After (post-Octane)
```js
// parent-component.js
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { action } from '@ember/object';

export default class ParentComponent extends Component {
  @tracked count = 0;

  @action plusOne() {
    this.count++;
  }
}

```
```hbs
{{!-- parent-component.hbs --}}
<ChildComponent @plusOne={{this.plusOne}} />
Count: {{this.count}}

```
```hbs
{{!-- child-component.hbs --}}
<button type="button" {{on "click" @plusOne}}>
  Click Me
</button>

```

</details>

But what we could put in the deprecation app:

### Scenario: `action` is passed a string

Before:
```hbs
<button type="button" {{action "plusOne"}}>
  Click Me
</button>
```

After

```hbs
<button type="button" {{on 'click' this.plusOne}}>
  Click Me
</button>
```
or, if `plusOne` is passed in as an argument 
```hbs
<button type="button" {{on 'click' @plusOne}}>
  Click Me
</button>
```

If the `plusOne` action is in an actions object, it needs to move out:

Before:
```js
import Component from '@glimmer/component';

export default class Demo extends Component {
    actions = {
        plusOne() {
           /* ... */ 
        }
    }
}
```
or
```js
import Component from '@ember/component';

export default class Demo extends Component {
    actions = {
        plusOne() {
           /* ... */ 
        }
    }
}
```
or
```js
import Component from '@ember/component';

export default Component.extend({
    actions: {
        plusOne() {
           /* ... */ 
        }
    }
})
```

After:
```js
import Component from '@glimmer/component';
import { action } from '@ember/object';

export default class Demo extends Component {
    @action
    plusOne() {
       /* ... */ 
    }
}
```

Note that `@action` is completely different from `(action)` or `{{action}}` (and is partly a motivator for deprecating `(action)` and `{{action}}`, to reduce ambiguity).

`@action` is binds the `this` on the method to the instance of the class. 

### Scenario: `action` is passed a function reference

Before:
```hbs
<SomeComponent @update={{action this.plusOne}} />
```

After

```hbs
<SomeComponent @update={{this.plusOne}} />
```

### Scenario: `action` is passed parameters

Before:
```hbs
<SomeComponent @update={{action this.plus 1}} />
```

After:
```hbs
<SomeComponent @update={{fn this.plus 1}} />
```

### Scenario: `action` is used with `mut` 

Before:
```hbs
<SomeComponent @update={{action (mut @value.property)}} />
```
After:
```js
// parent.js
import Component from '@glimmer/component';
import { action } from '@ember/object';

export default class SomeComponent extends Component {
    @action
    handleUpdate(value) {
        this.args.property = value; 
    }
}
```
```hbs
{{! parent.hbs }}
<SomeComponent @update={{this.handleUpdate}} />
```

Related, [Combining function arguments with action functions](https://guides.emberjs.com/release/components/component-state-and-actions/#toc_combining-arguments-and-actions)

### Related: `send`

When removing `(action)` or `{{action}}` with a _string_ name, you'll also need to verify that there are no [`send`](https://api.emberjs.com/ember/5.6/classes/Component/methods/send?anchor=send) calls with that same string.

## How We Teach This

The guides already cover how to invoke functions in the modern way.

Remove: https://api.emberjs.com/ember/5.6/classes/Ember.Templates.helpers/methods/action?anchor=action

## Drawbacks

Older code will stop working once the deprecated code is removed.

## Alternatives

- adding an import so folks can keep using action in gjs.
  I don't think we should do this because we want to clean up antiquated patterns, rather than encourage their continued existence.

## Unresolved questions

- Could there be a codemod?
  _Potentially_ for action usage that references `this.properties`. For string actions, it's impossible.

