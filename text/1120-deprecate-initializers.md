---
stage: accepted
start-date: 2025-07-02T00:00:00.000Z # In format YYYY-MM-DDT00:00:00.000Z
release-date:
release-versions:
teams: # delete teams that aren't relevant
  - framework
  - learning
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1120 
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

# Deprecate Application Initializers and Instance Initializers

## Summary

This RFC proposes deprecating Ember's application initializers and instance initializers in favor of explicit application lifecycle management. The current initializer system will be replaced with direct access to application and instance objects during the boot process, enabling async/await support and providing more explicit control over application setup.

> [!NOTE] 
> This RFC assumes usage of the v2 app blueprint, via `@ember/app-blueprint`

[Here is a demo app](https://github.com/NullVoxPopuli/ember-initializers-demo/commit/f57e30c0cee5e83794fa71961a8d091ededaf292)

## Motivation

The current initializer system has several limitations that make it difficult to work with in modern JavaScript environments:

1. **No async/await support**: Initializers cannot be awaited, making asynchronous setup operations difficult to coordinate and debug.

2. **Implicit execution order**: The `before` and `after` properties create implicit dependencies that are hard to reason about and can lead to subtle ordering bugs.

3. **Hidden lifecycle**: The automatic execution of initializers makes the application boot process opaque and harder to debug.

4. **Limited access to application state**: Initializers receive only the application or instance object, without access to other important lifecycle information.

5. **Testing complexity**: The automatic execution of initializers during testing requires complex workarounds to avoid side effects.

The replacement functionality provides explicit control over the application lifecycle by allowing developers to write setup code directly in their application entry point, with full access to modern JavaScript async/await patterns.

## Transition Path

### Current Initializer Usage Patterns

Application initializers are currently defined in `app/initializers/` and run during application boot:

```js
// app/initializers/session.js
export default {
  name: 'session',
  initialize(application) {
    application.inject('route', 'session', 'service:session');
  }
};
```

Instance initializers are defined in `app/instance-initializers/` and run during instance creation:

```js
// app/instance-initializers/current-user.js
export default {
  name: 'current-user',
  initialize(applicationInstance) {
    let session = applicationInstance.lookup('service:session');
    return session.loadCurrentUser();
  }
};
```

### New Explicit Lifecycle Pattern

This would replace the contents of the script tag in the existing `@ember/app-blueprint`:
```html
<script type="module">
  import Application from './app/app';
  import environment from './app/config/environment';

  Application.create(environment.APP);
</script>
```

However! We will not change the default from the above, as the simple case is the most common.

> [!IMPORTANT]
> The above 3 line entrypoint for ember applications will remain the default. This RFC is primarily about updates to documentation showing how we can achieve a new way of "initializers" while also slimming the default experience for new projects.

The new pattern provides explicit control over application lifecycle in the main application entry point:

```js
import Application from '#app/app.ts';
import environment from '#app/config.ts';

let app = Application.create({
  ...environment.APP,
  autoboot: false,
});

// anything running here with access to `app`
// is equivelent to classic "initializers"
//
// These can be awaited, which was not possible before
console.log(app);

await app.boot();
let instance = await app.buildInstance();
await instance.boot();
await instance.visit('/');

// anything running here with access to `instance`
// is equivelent to classic "instance initializers"
//
// These can be awaited, which was not possible before
console.log(instance);
```

### To share initialization with tests

Developers can extract the contents of the script tag to a `boot.ts` or `boot.js`, with the same contents.

> [!DANGER]
> Since initializers would no longer be tied to the application instance itself, initializers would not normally run during tests.

To have initializers run during tests, a developer would want to edit their in-test-copy of `setupApplicationTest` to run the initializers there, for example:

```ts
function setupApplicationTest(hooks: NestedHooks, options?: SetupTestOptions) {
    upstreamSetupApplicationTest(hooks, options);

    hooks.beforeEach(function () {
        let instance = this.owner;
        runInstanceInitializer(instance)
    });

    // Additional setup for application tests can be done here.
    //
    // For example, if you need an authenticated session for each
    // application test, you could do:
    //
    // hooks.beforeEach(async function () {
    //   await authenticateSession(); // ember-simple-auth
    // });
    //
    // This is also a good place to call test setup functions coming
    // from other addons:
    //
    // setupIntl(hooks, 'en-us'); // ember-intl
    // setupMirage(hooks); // ember-cli-mirage
}
```


### Migration Steps for Common Use Cases

#### 1. Service Registration

**Before:**
```js
// app/initializers/register-websocket.js
export default {
  name: 'register-websocket',
  initialize(application) {
    application.register('service:websocket', WebSocketService);
  }
};
```

**After:**
```js
// In main.js or index.html
let app = Application.create({ ...environment.APP, autoboot: false });

app.register('service:websocket', WebSocketService);

await app.boot();
// ... rest of boot process
```

#### 2. Asynchronous Setup

**Before (not possible):**
```js
// app/initializers/async-setup.js
export default {
  name: 'async-setup',
  initialize(application) {
    // Cannot await here - initializers don't support async
    fetchConfigFromServer().then(config => {
      application.register('config:remote', config);
    });
  }
};
```

**After:**
```js
// In main.js or index.html
let app = Application.create({ ...environment.APP, autoboot: false });

// Now we can await asynchronous setup
let config = await fetchConfigFromServer();

await app.boot();
// ... rest of boot process
```

#### 3. Instance-level Setup

**Before:**
```js
// app/instance-initializers/load-user.js
export default {
  name: 'load-user',
  initialize(applicationInstance) {
    let session = applicationInstance.lookup('service:session');
    return session.loadCurrentUser();
  }
};
```

**After:**
```js
// In main.js or index.html
let app = Application.create({ ...environment.APP, autoboot: false });
await app.boot();
let instance = await app.buildInstance();

// Instance-level setup with full async/await support
let session = instance.lookup('service:session');
await session.loadCurrentUser();

await instance.boot();
await instance.visit('/');
```

### Ecosystem Implications

- **ember-load-initializers**: This addon will be README-deprecated as initializer discovery will no longer be needed -- the README will show the new way to run initializers if they are needed.
- **Blueprints**: Ember CLI blueprints should be updated to remove use of initializers 
- **Addon ecosystem**: Addons that provide initializers will need to document the migration path -- which is to import the initializer from the addon, and place it in the script tag in the appropriate place.

## How We Teach This

This change represents a significant shift in how Ember applications are structured, requiring updates across multiple learning resources:

### Guides Updates

The "Applications and Instances" section of the guides will need to be rewritten to:
- Remove documentation for initializers and instance initializers
- Add comprehensive documentation for explicit application lifecycle management
  - this is needed anyway because it's not described anywhere
- Provide migration examples for common initializer patterns
- Explain the benefits of explicit control and async/await support

### API Documentation

- Mark initializer-related APIs as deprecated

## Drawbacks

- Large applications with many initializers will require a little migration effort
- Addon authors will need to update their libraries and documentation -- initializers from addons would no longer run automatically

## Alternatives

### Option 1: Explicit Initializer Registration

Keep the initializer concept but require explicit registration:

```js
// In main.js
import sessionInitializer from './initializers/session';
import userInitializer from './instance-initializers/user';

let app = Application.create({ ...environment.APP, autoboot: false });
app.registerInitializer(sessionInitializer);
app.registerInstanceInitializer(userInitializer);
```

**Pros**: This is the closest to the existing behavior with the least amount of boilerplate. 
**Cons**: Still maintains the complexity of the initializer system, and folks that don't use it will still have to pay for its existence. 

### Option 2: Do Nothing

Continue with the current initializer system.

**Pros**: No migration effort required
**Cons**: Continues to limit developers from using modern JavaScript patterns and maintains the current complexity and debugging difficulties

## Unresolved questions

n/a

## References

- async initializers were first asked about in [2013](https://discuss.emberjs.com/t/initializers-that-return-promises/2782)  (maybe earlier?)
- previous RFC was attempted, [RFC#572](https://github.com/emberjs/rfcs/pull/572)
