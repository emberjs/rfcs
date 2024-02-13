---
stage: accepted
start-date: 2024-02-13T00:00:00.000Z
release-date:
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1006
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

# Deprecate `(action)` template helper 

## Summary

The `(action)` template helper was common pre-Octane. Now that we have native classes and the `{{on}}` modifier, we no longer need to use `(action)` 

## Motivation

Remove legacy code with confusing semantics.

## Transition Path

This was written in the [Octave vs Classic cheatsheet](https://ember-learn.github.io/ember-octane-vs-classic-cheat-sheet/#component-properties__ddau)

That content here:

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

## How We Teach This

The guides already cover how to invoke functions in the modern way.

Remove: https://api.emberjs.com/ember/5.6/classes/Ember.Templates.helpers/methods/action?anchor=action

## Drawbacks

Older code will stop working once the deprecated code is removed.

## Alternatives

## Unresolved questions

