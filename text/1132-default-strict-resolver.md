---
stage: accepted
start-date: 2025-08-13T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams:
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1132
project-link:
suite: 
---

# Default Strict Resolver

## Summary

This RFC proposes shipping a built-in strict resolver as the default in Ember applications, replacing the current `ember-resolver` package. The strict resolver uses explicit module registration through `import.meta.glob('...', { eager: true })` (or manually) instead of dynamic string-based lookups, providing better tree-shaking, build-time optimization, and improved developer experience with static analysis tools.

## Motivation

The current `ember-resolver` requires that an `Application` has a modulePrefix, and that every resolver registration begins with that `modulePrefix`. As we start to move towards strict ESM applications, this requirement is unneeded. Previously, if folks wanted to use `import.meta.glob()` in their vite apps (or test-apps (or minimal apps)), they would have to iterate the result of `import.meta.glob` to prepend the `modulePrefix` -- which is entirely boilerplate.

> [!NOTE]
> The strict resolver does not affect apps using broccoli or `@embroider/virtual/compat-modules`

## Detailed design

The strict resolver will be built into Ember applications and imported from `@ember/application` instead of requiring a separate `ember-resolver` package. Applications will register modules explicitly using a `modules` property containing import map objects.

Example:

```javascript
// Before | with ember-resolver
import Application from '@ember/application';
import Resolver from 'ember-resolver';
import config from 'my-app/config/environment';

export default class App extends Application {
  modulePrefix = config.modulePrefix;
  podModulePrefix = config.podModulePrefix;
  Resolver = Resolver.withModules({
    [`${config.modulePrefix}/router`]: { default: Router },
    [`${config.modulePrefix}/services/current-user`]: { default: CurrentUserService },
    // potentially thousands of manual entries
    // and all imports need to be at the top of the file as well -- doubling the lines per resolver entries
  });
}
```

```javascript
// After: 
import Application from '@ember/application';
import Router from './router';
import config from 'my-app/config/environment';

export default class App extends Application {
  modules = {
    './router': { default: Router },
    './services/current-user': { default: CurrentUserService },
    
    // author-time is easy
    ...import.meta.glob('./services/**/*', { eager: true }),
    ...import.meta.glob('./routes/*', { eager: true }),
    ...import.meta.glob('./controllers/*', { eager: true }),
    ...import.meta.glob('./components/**/*', { eager: true }),
    ...import.meta.glob('./templates/**/*', { eager: true }),
    ...import.meta.glob('./helpers/*', { eager: true }),
    ...import.meta.glob('./modifiers/*', { eager: true }),
  };
}
```

### Module Registration Format

The `modules` object maps module paths to either:
1. Module objects with named exports: `{ default: ExportableType, namedExport: AnotherType }`
2. Direct exports for default-only modules: `ExportableType`

```javascript
modules = {
  // Full module object
  './services/current-user': { default: CurrentUserService },
  './components/user-avatar': { default: UserAvatarComponent },
  
  // Shorthand for default exports
  './services/session': SessionService,
  
  // Automatic splat via import.meta.glob
  ...import.meta.glob('./services/**/*', { eager: true }),
}
```


## How we teach this

API Docs will explain the above usages.
We aren't ready to add this to the guides as the community may not be ready for "minimal app" (which may need to be a different app blueprint from the current `@ember/app-blueprint`).

## Appendix


Libraries can provide helper functions for registering their modules, services, etc:

```javascript
// Library provides a registry builder
import { buildRegistry } from 'ember-strict-application-resolver/build-registry';
import libraryRegistry from 'some-library/registry';

export default class App extends Application {
  modules = {
    ...import.meta.glob('./services/**/*', { eager: true }),
    ...libraryRegistry(), // Library-provided modules
    './services/custom': CustomService,
  };
}
```

See: [the polyfill](https://github.com/NullVoxPopuli/ember-strict-application-resolver?tab=readme-ov-file#buildregistry) for more information.

## Drawbacks

n/a - folks can keep using ember-resolver. ember-resolver is _not_ deprecated.

## Unresolved questions

n/a