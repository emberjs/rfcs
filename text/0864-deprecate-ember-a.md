---
stage: accepted
start-date: 2022-11-22T00:00:00.000Z
release-date:
release-versions:
teams:
  - data
  - framework
  - learning
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/864
project-link:
---

# Deprecate Ember.A()

## Summary

Remove `Ember.A()` as the functionality it provides is no longer needed in the post octane reactivity system and with the removal of array prototype extensions in #848.

## Motivation

`Ember.A()` provides a way for addon authors or application owners who disable array prototype extensions to add reactivity to arrays (through `pushObject()`, `removeObject()`) and to access prototype extension convenience methods such as `filterBy()` on an any array-like object. Both of these paradigms have been deprecated in Ember and `Ember.A()` should follow suit as it can break things in unexpected ways and interferes with progress in [Ember](https://github.com/emberjs/ember.js/blob/4339725976299b24c69fb9dfbf13d18bf9917130/packages/@ember/-internals/utils/lib/ember-array.ts) and [Ember Data](https://github.com/emberjs/data/blob/47a71ca1538ba9e2d7dfa01bf048a2db897bdf5f/packages/store/addon/-private/record-arrays/identifier-array.ts#L381-L401) where special care must be taken to prevent or compensate for usage.

Along with it's usefulness `Ember.A()` has a significant API gotcha in that it doesn't return a new instance of the array-like object it is passed, it modifies that object and returns is such that `A(array) === array` this can create the dreaded spooky-action-at-a-distance effect where suddenly an object which was `Array` can become an `Ember.NativeArray`. An example of this is using `{{includes "Zoey" this.mascots}}` from [ember-composable-helpers](https://github.com/DockYard/ember-composable-helpers#includes) will modify `this.mascots` and add `lastObject` to the API. Re-ordering the template code at a later date will result in a mysterious failure that can be extremely frustrating to understand.

```js
export default class WeirdArray extends Component {
  mascots = ['Tomster', 'Zoey'];

  get lastMascot() {
    return this.mascots.lastObject;
  }

  <template>
    {{#if (includes "Zoey" this.mascots)}}
      Zoey Rules!
    {{/if}}
    Last Mascot: {{this.lastMascot}}
  </template>
}
```

Deprecating `Ember.A()` will provide a signal to the addon community to move away from both array prototype extensions and array reactivity which requires using the Emberism `pushObject`/`removeObject` to track updates. 

Now that array prototype extensions have been deprecated it is quite possible that usage of `Ember.A()` will increase significantly as a defensive coding strategy to maintain access to the family of methods Ember adds to all arrays. This should be immediately discouraged as `Ember.A()` is not a long term viable solution and other better alternatives exist.

## Transition Path

### Replacement APIs

As discussed in #848 better alternatives exist for the APIs provided by `Ember.A()`:

> For convenient methods like `filterBy`, `compact`, `sortBy` etc., the replacement functionalities already exist either through native array methods or utility libraries like [lodash](https://lodash.com), [Ramda](https://ramdajs.com), etc.

>For mutation methods (like `pushObject`, `removeObject`) or observable properties (like `firstObject`, `lastObject`) participating in the Ember classic reactivity system, the replacement functionalities also already exist in the form of immutable update style with tracked properties like `@tracked someArray = []`, or through utilizing `TrackedArray` from `tracked-built-ins`.

### Clearly Define Idiomatic Path

Deprecating `Ember.A()` will provide a clear signal to move away from this construct and rely on these more robust modern alternatives.

### Transitional Feature Flags

Transition away from `Ember.A()` should be aided by a pair of feature flags:

#### Wrap Passed Value

Instead of returning the passed value `Ember.A()` would return a native proxy to wrap the original object without modifying it. This would remove the most confusing part of the API and allow apps to opt in early and clear any unexpected issues before `Ember.A()` is fully removed.

#### Noop Ember.A

Currently if array prototype extensions is enabled `Ember.A()` is a `noop` and returns the passed value unmodified. It should be possible for application developers to turn off array prototype extensions and maintain this behavior of `Ember.A()`. This will make bugs easier to find and allow applications to more quickly prevent prototype pollution of the `Array` API before `Ember.A()` is fully removed.


## How We Teach This

`Ember.A()` is not covered in the guides, but it is present in [many](https://emberobserver.com/code-search?codeQuery=import%20%7B%20A%20%7D%20from%20%27%40ember%2Farray%27%3B) [many](https://emberobserver.com/code-search?codeQuery=Ember.A) addons. 

Usage of convenience APIs can be easily replaced using the [no-array-prototype-extensions](https://github.com/ember-cli/eslint-plugin-ember/blob/4820f0fb286e40872a77d687618be56999f23704/docs/rules/no-array-prototype-extensions.md) codemod.

For reactivity addon authors will need to rely forcing tracking as `this.mascots = [...this.mascots, 'Rock & Roll']` or through utilizing `TrackedArray` from [tracked-built-ins](https://github.com/tracked-tools/tracked-built-ins).

## Drawbacks

`Ember.A()` has been a huge part of building Ember Addons for a long time and it is present in many popular addons. Removing this feature will cause a significant amount of churn in otherwise stable addons and require a significant community effort to replace.

## Alternatives

### Change Confusing API

Instead of modifying the passed value `Ember.A()` could return a new object which contains a reference to the original value. This would avoid the `A(array) === array` issue and make it easier to detect if an object had been passed through `A()` and to work with the original value if necessary. This would still be a breaking change as many apps (often unknowingly) rely on the behavior of an object having been passed through `A()`.

## Unresolved questions
