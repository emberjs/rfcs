---
stage: accepted
start-date: 2020-11-28T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/686
project-link:
---

# Deprecate Old Manager Capabilities

## Summary

Deprecate older capabilities versions from the various manager APIs.

## Motivation

In the 3.x cycle, Ember introduced a series of new low-level APIs for managing
template constructs:

- [Component managers](https://github.com/emberjs/rfcs/blob/master/text/0213-custom-components.md)
- [Modifier managers](https://github.com/emberjs/rfcs/blob/master/text/0373-Element-Modifier-Managers.md)
- [Helper managers](https://github.com/emberjs/rfcs/blob/master/text/0625-helper-managers.md)

These APIs were expected to evolve more quickly than the higher level APIs they
enabled, and they have in fact done so. They do this via the `capabilities`
mechanism, where users can specify a version of capabilities that they want.
This allows Ember to change these APIs as needs evolve, as long as the prior
capabilities can still be maintained.

Maintaining the prior capabilities versions has a cost however in terms of
maintenance burden, and sometimes requires us to keep around a decent amount of
extra code or internal features. Deprecating these versions for the next major
release will help clean up code internally overall.

## Transition Path

Users should update to the most recent manager versions, which will not be
deprecated. The versions which are being deprecated include:

- Component Managers
  - 3.4
- Modifier Managers
  - 3.13

The versions that are still supported are:

- Component Managers
  - 3.13
- Modifier Managers
  - 3.22
- Helper Managers
  - 3.23

## How We Teach This

In general, the guides won't need to be updated as there isn't guide material
for these manager APIs. We should update the API docs for them to remove the
deprecated capabilities versions. In addition, we should add deprecation guides
for each of the deprecated versions.

Guides as follows.

### Component Managers

#### 3.4

Any component managers using the `3.4` capabilities should update to the most
recent component capabilities that are available, currently `3.13`. In `3.13`,
the only major change is that update hooks are no longer called by default. If
you need update hooks, use the `updateHook` capability:

```js
capabilities({
  updateHook: true,
});
```

### Modifier Managers

#### 3.13

Any modifier managers using the `3.13` capabilities should update to the most
recent modifier capabilities, currently `3.22`. In `3.22`, the major changes
are:

1. The modifier definition, associated via `setModifierManager` is passed
   directly to `create`, rather than a factory wrapper class. Previously, you
   would access the class via the `class` property on the factory wrapper:

   ```js
   // before
   class CustomModifierManager {
     capabilities = capabilities('3.13');

     createModifier(Definition, args) {
       return new Definition.class(args);
     }
   }
   ```

   This can be updated to use the definition directly:

   ```js
   // after
   class CustomModifierManager {
     capabilities = capabilities('3.22');

     createModifier(Definition, args) {
       return new Definition(args);
     }
   }
   ```

2. Args are both lazy and autotracked by default. This means that in order to
   track an argument value, you must actually use it in your modifier. If you do
   not, the modifier will not update when the value changes.

   If you still need the modifier to update whenever a value changes, even if it
   was not used, you can manually access every value in the modifiers
   `installModifier` and `updateModifier` lifecycle hooks:

   ```js
   function consumeArgs(args) {
     for (let key in args.named) {
       // consume value
       args.named[key];
     }

     for (let i = 0; i < args.positional.length; i++) {
       // consume value
       args.positional[i];
     }
   }

   class CustomModifierManager {
     capabilities = capabilities('3.22');

     installModifier(bucket, element, args) {
       consumeArgs(args);

       // ...
     }

     updateModifier(bucket, args) {
       consumeArgs(args);

       // ...
     }
   }
   ```

   In general this should be avoided, however, and users who are writing
   modifiers should instead use the value if they want it to be tracked by the
   modifier.

## Drawbacks

- There will be some minor churn in the ecosystem as managers update, but this
  is outweighed by the lowered maintenance burden in general for this
  functionality.

## Alternatives

- Maintian capabilities versions indefinitely. This is not really feasible, as
  more and more changes will likely happen over time and eventually it will
  result in a large maintenance burden.
