---
stage: ready-for-release
start-date: 2025-07-15T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1041'
project-link:
suite:
---

# Deprecate TargetActionSupport

## Summary

Deprecate `send` and the corresponding TargetActionSupport.

## Motivation

These are legacy patterns that are no longer recommended in modern Ember code. The primary way to interact with this system was the now-deprecated `{{action}}` modifier/helper. The modern approach is to use standard class methods (optionally decorated with `@action`) and to pass functions directly, following the Data Down, Actions Up (DDAU) pattern.

See also: [Target Action Support deprecation guide](https://deploy-preview-1410--ember-deprecations.netlify.app/deprecations/v6.x/#target-action-support)

## Detailed design

This RFC proposes to deprecate and remove the following:

- `TargetActionSupport` mixin, specifically the `triggerAction` method
- `ActionHandler` mixin, specifically the `send` method
- `ActionSupport` mixin, specifically the `send` method

These mixins and methods are legacy patterns that predate modern Ember features like native classes, tracked properties, and Glimmer components. Their primary use case was to support the `{{action}}` helper/modifier, which has already been deprecated.

### Deprecation Process

1. Mark the affected mixins and methods as deprecated in the next minor release, with clear deprecation warnings and migration guides.
2. Update documentation to reflect the deprecation and recommend alternatives (such as closure actions, service injection, or direct method calls).
3. Remove the deprecated APIs in a future major release.

### Migration Path

- For `send` and `triggerAction`, recommend direct method invocation or using closure actions.
- Provide codemods or lint rules to help users identify and migrate away from these patterns.


### Example Migration

**Before:**

```js
// Using send
this.send('doSomething');
```

**After:**

```js
// Direct method call
this.doSomething();
```

**Before:**

```js
// Using triggerAction
this.triggerAction({ action: 'save' });
```

**After:**

```js
// Direct method call or closure action
this.save();
// or pass a closure action
@action
save() { /* ... */ }
```


### Deprecation Message

When these APIs are used, emit a deprecation warning:

```
The use of `send`/`triggerAction`/target action support is deprecated. See https://deploy-preview-1410--ember-deprecations.netlify.app/deprecations/v6.x/#target-action-support for migration details.
```

### Ecosystem Implications

- **Lint rules:** Add rules to `ember-template-lint` and `eslint-plugin-ember` to flag usage of these APIs.
- **Features replaced/obsolete:** The `{{action}}` helper/modifier and related mixins are already deprecated or replaced by modern patterns.
- **Ember Inspector:** Remove or update any features that rely on these APIs.
- **Server-side Rendering, Ember Engines, Addon Ecosystem, IDE Support:** No direct impact, but documentation and blueprints should be updated to avoid these patterns.

---


## How we teach this

This deprecation is a continuation of Ember's modernization, moving away from legacy patterns toward direct method calls and closure actions. The recommended terminology is "direct method invocation" and "closure actions".

**Guides and Documentation:**
- Remove references to `send`, `triggerAction`, and the affected mixins from the Ember Guides and API docs.
- Add a deprecation guide (see above) and migration examples.
- Update blog posts and tutorials to avoid these patterns.

**For Existing Users:**
- Announce the deprecation in release notes and community channels.
- Provide codemods, lint rules, and clear migration paths.

**For New Users:**
- Ensure new learning materials do not mention or rely on these legacy APIs.

---


## Drawbacks

- Some legacy apps and addons may still rely on these patterns, so deprecation and removal could require migration effort.
- Removing these APIs may break code that uses dynamic action dispatching via `send` or `triggerAction`.
- There is a risk of fragmenting documentation and learning resources during the transition period.

---


## Alternatives

- Do nothing: Continue to support these legacy APIs, but this increases maintenance burden and hinders Ember's modernization.
- Deprecate but do not remove: This avoids breaking changes but leaves deprecated code in the framework indefinitely.
- Replace with a new abstraction: No clear need, as modern JavaScript and Ember patterns already provide better alternatives.

**Prior Art:**
Other frameworks (React, Vue, Angular) encourage direct method calls or use of closures for event handling, rather than a dynamic action dispatch system.

---


## Unresolved questions

- Are there any edge cases or internal uses of these mixins that need special handling?
- Should we provide an official codemod for migration, or rely on lint rules and manual refactoring?
- Are there any third-party addons that would be significantly impacted and need outreach or support?
