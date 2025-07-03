---
stage: accepted
start-date: 2025-07-03T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams:
  - framework
  - learning
  - typescript
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
suite: 
---

# Add Resources as a Reactive Primitive

## Summary

Resources are a reactive primitive that enables managing stateful processes with cleanup logic as reactive values. They unify concepts like custom helpers, modifiers, and services by providing a consistent pattern for expressing values that have lifecycles and may require cleanup when their owner is destroyed.

## Motivation

Ember's current Octane programming model provides excellent primitives for reactive state (`@tracked`), declarative rendering (templates), and component lifecycle, but it lacks a unified primitive for managing stateful processes that need cleanup. This gap leads to several problems:

### Current Pain Points

**1. Scattered Lifecycle Management**
Today, managing stateful processes requires spreading setup and cleanup across component lifecycle hooks:

```js
export default class TimerComponent extends Component {
  @tracked time = new Date();
  timer = null;

  constructor() {
    super(...arguments);
    this.timer = setInterval(() => {
      this.time = new Date();
    }, 1000);
  }

  willDestroy() {
    if (this.timer) {
      clearInterval(this.timer);
    }
  }
}
```

This pattern has several issues:
- Setup and cleanup logic are separated across the component lifecycle
- It's easy to forget cleanup, leading to memory leaks
- The logic is tied to component granularity
- Testing requires instantiating entire components

**2. Reactive Cleanup Complexity**
When tracked data changes, manually managing cleanup of dependent processes is error-prone:

```js
export default class DataLoader extends Component {
  @tracked url;
  controller = null;

  @cached
  get request() {
    // Need to manually handle cleanup when URL changes
    this.controller?.abort();
    this.controller = new AbortController();
    
    return fetch(this.url, { signal: this.controller.signal });
  }

  willDestroy() {
    this.controller?.abort();
  }
}
```

**3. Code Organization and Reusability**
Business logic becomes tightly coupled to components, making it hard to:
- Extract and reuse patterns across components
- Test logic in isolation
- Compose smaller pieces into larger functionality

**4. Lack of Unified Abstraction**
Ember has several overlapping concepts that solve similar problems:
- Custom helpers for derived values
- Modifiers for DOM lifecycle management  
- Services for shared state
- Component lifecycle hooks for cleanup

Each uses different APIs and patterns, creating cognitive overhead and preventing a unified mental model.

### What Resources Solve

Resources provide a unified primitive that:

1. **Co-locates setup and cleanup logic** in a single function
2. **Automatically handles reactive cleanup** when dependencies change
3. **Enables fine-grained composition** independent of component boundaries
4. **Provides a consistent abstraction** for all stateful processes with cleanup
5. **Improves testability** by separating concerns from component lifecycle

Resources allow developers to model any stateful process as a reactive value with an optional cleanup lifecycle, unifying concepts across the framework while maintaining Ember's declarative, reactive programming model.

## Detailed design

### Overview

A **resource** is a reactive function that represents a value with lifecycle and optional cleanup logic. Resources are created using the `resource()` function and automatically manage their lifecycle through Ember's existing destroyable system.

```js
import { resource } from '@ember/resources';
import { cell } from '@glimmer/tracking';

const Clock = resource(({ on }) => {
  const time = cell(new Date());
  
  const timer = setInterval(() => {
    time.set(new Date());
  }, 1000);

  on.cleanup(() => clearInterval(timer));

  return time;
});
```

### Core API

The `resource()` function takes a single argument: a function that receives a ResourceAPI object and returns a reactive value.

```ts
interface ResourceAPI {
  on: {
    cleanup: (destructor: () => void) => void;
  };
  use: <T>(resource: T) => ReactiveValue<T>;
  owner: Owner;
}

type ResourceFunction<T> = (api: ResourceAPI) => T;

function resource<T>(fn: ResourceFunction<T>): Resource<T>
```

### Resource Creation and Usage

Resources can be used in several ways:

**1. In Templates (as helpers)**
```js
const Clock = resource(({ on }) => {
  const time = cell(new Date());
  const timer = setInterval(() => time.set(new Date()), 1000);
  on.cleanup(() => clearInterval(timer));
  return time;
});

// In template
<template>Current time: {{Clock}}</template>
```

**2. With the `@use` decorator**
```js
import { use } from '@ember/resources';

export default class MyComponent extends Component {
  @use clock = Clock;
  
  <template>Time: {{this.clock}}</template>
}
```

**3. With the `use()` function**
```js
export default class MyComponent extends Component {
  clock = use(this, Clock);
  
  <template>Time: {{this.clock.current}}</template>
}
```

**4. Manual instantiation (for library authors)**
```js
const clockBuilder = resource(() => { /* ... */ });
const owner = getOwner(this);
const clockInstance = clockBuilder.create();
clockInstance.link(owner);
const currentTime = clockInstance.current;
```

### Resource API Details

**`on.cleanup(destructor)`**

Registers a cleanup function that will be called when the resource is destroyed. This happens automatically when:
- The owning context (component, service, etc.) is destroyed
- The resource re-runs due to tracked data changes
- The resource is manually destroyed

```js
const DataLoader = resource(({ on }) => {
  const controller = new AbortController();
  const state = cell({ loading: true });

  on.cleanup(() => controller.abort());

  fetch('/api/data', { signal: controller.signal })
    .then(response => response.json())
    .then(data => state.set({ loading: false, data }))
    .catch(error => state.set({ loading: false, error }));

  return state;
});
```

**`use(resource)`**

Allows composition of resources by consuming other resources with proper lifecycle management:

```js
const Now = resource(({ on }) => {
  const time = cell(new Date());
  const timer = setInterval(() => time.set(new Date()), 1000);
  on.cleanup(() => clearInterval(timer));
  return time;
});

const FormattedTime = resource(({ use }) => {
  const time = use(Now);
  return () => time.current.toLocaleTimeString();
});
```

**`owner`**

Provides access to the Ember owner for dependency injection:

```js
const UserSession = resource(({ owner }) => {
  const session = owner.lookup('service:session');
  const router = owner.lookup('service:router');
  
  return () => ({
    user: session.currentUser,
    route: router.currentRouteName
  });
});
```

### Resource Lifecycle

1. **Creation**: When a resource is first accessed, its function is invoked
2. **Reactivity**: If the resource function reads tracked data, it will re-run when that data changes
3. **Cleanup**: Before re-running or when destroyed, all registered cleanup functions are called
4. **Destruction**: When the owning context is destroyed, the resource and all its cleanup functions are invoked

### Type Definitions

```ts
interface ResourceAPI {
  on: {
    cleanup: (destructor: () => void) => void;
  };
  use: <T>(resource: T) => ReactiveValue<T>;
  owner: Owner;
}

type ResourceFunction<T> = (api: ResourceAPI) => T;

interface Resource<T> {
  create(): ResourceInstance<T>;
}

interface ResourceInstance<T> {
  current: T;
  link(context: object): void;
}

function resource<T>(fn: ResourceFunction<T>): Resource<T>;
function use<T>(context: object, resource: Resource<T>): ReactiveValue<T>;
function use<T>(resource: Resource<T>): PropertyDecorator;
```

### Integration with Existing Systems

**Helper Manager Integration**

Resources integrate with Ember's existing helper manager system. The `resource()` function returns a value that can be used directly in templates through the helper invocation syntax.

**Destroyable Integration**

Resources automatically integrate with Ember's destroyable system via `associateDestroyableChild()`, ensuring proper cleanup when parent contexts are destroyed.

**Ownership Integration**

Resources receive the owner from their parent context, enabling dependency injection and integration with existing Ember services and systems.

### Example Use Cases

**1. Data Fetching**
```js
const RemoteData = resource(({ on }) => {
  const state = cell({ loading: true });
  const controller = new AbortController();

  on.cleanup(() => controller.abort());

  fetch(this.args.url, { signal: controller.signal })
    .then(response => response.json())
    .then(data => state.set({ loading: false, data }))
    .catch(error => state.set({ loading: false, error }));

  return state;
});
```

**2. WebSocket Connection**
```js
const WebSocketConnection = resource(({ on, owner }) => {
  const notifications = owner.lookup('service:notifications');
  const socket = new WebSocket('ws://localhost:8080');
  
  socket.addEventListener('message', (event) => {
    notifications.add(JSON.parse(event.data));
  });

  on.cleanup(() => socket.close());

  return () => socket.readyState;
});
```

**3. DOM Event Listeners**
```js
const WindowSize = resource(({ on }) => {
  const size = cell({
    width: window.innerWidth,
    height: window.innerHeight
  });

  const updateSize = () => size.set({
    width: window.innerWidth,
    height: window.innerHeight
  });

  window.addEventListener('resize', updateSize);
  on.cleanup(() => window.removeEventListener('resize', updateSize));

  return size;
});
```

### Implementation Notes

The implementation will build upon Ember's existing infrastructure:

- **Helper Manager**: Resources use the existing helper manager system for template integration
- **Destroyable System**: Automatic lifecycle management through existing destroyable primitives
- **Tracking System**: Resources participate in Glimmer's autotracking system
- **Owner System**: Resources receive owners for dependency injection

The core implementation involves:
1. A helper manager that creates and manages resource instances
2. Integration with `@glimmer/tracking` for reactivity
3. Integration with `@ember/destroyable` for cleanup
4. TypeScript definitions for proper type inference

## How we teach this

### Terminology and Naming

The term **"resource"** was chosen because:

1. It accurately describes something that requires management (like memory, network connections, timers)
2. It's already familiar to developers from other contexts (system resources, web resources)
3. It emphasizes the lifecycle aspect - resources are acquired and must be cleaned up
4. It unifies existing concepts without deprecating them

**Key terms:**
- **Resource**: A reactive function that manages a stateful process with cleanup
- **Resource Function**: The function passed to `resource()` that defines the behavior
- **Resource Instance**: The instantiated resource tied to a specific owner
- **Resource API**: The object passed to resource functions containing `on`, `use`, and `owner`

### Teaching Strategy

**1. Progressive Enhancement of Existing Patterns**

Introduce resources as a natural evolution of patterns developers already know:

```js
// Familiar pattern
export default class extends Component {
  @tracked time = new Date();
  
  constructor() {
    super(...arguments);
    this.timer = setInterval(() => {
      this.time = new Date();
    }, 1000);
  }
  
  willDestroy() {
    clearInterval(this.timer);
  }
}

// Resource pattern
const Clock = resource(({ on }) => {
  const time = cell(new Date());
  const timer = setInterval(() => time.set(new Date()), 1000);
  on.cleanup(() => clearInterval(timer));
  return time;
});
```

**2. Emphasize Problem-Solution Fit**

Lead with the problems resources solve:
- "Have you ever forgotten to clean up a timer?"
- "Want to reuse this data loading logic across components?"
- "Need to test stateful logic without rendering components?"

**3. Start with Simple Examples**

Begin with straightforward use cases before introducing composition:

```js
// Start here: Simple timer
const Timer = resource(({ on }) => {
  let count = cell(0);
  let timer = setInterval(() => count.set(count.current + 1), 1000);
  on.cleanup(() => clearInterval(timer));
  return count;
});

// Then: Data fetching
const UserData = resource(({ on }) => {
  // ... fetch logic
});

// Finally: Composition
const Dashboard = resource(({ use }) => {
  const timer = use(Timer);
  const userData = use(UserData);
  return () => ({ time: timer.current, user: userData.current });
});
```

### Documentation Strategy

**Ember Guides Updates**

1. **New Section**: "Working with Resources"
   - When to use resources vs components/services/helpers
   - Basic patterns and examples
   - Composition and testing

2. **Enhanced Sections**: 
   - Update "Managing Application State" to include resources
   - Add resource examples to "Handling User Interaction"
   - Include resource patterns in "Working with Data"

**API Documentation**

- Complete API reference for `resource()`, `use()`, and ResourceAPI
- TypeScript definitions with comprehensive JSDoc comments
- Interactive examples for each major use case

**Migration Guides**

- How to convert class-based helpers to resources
- Converting component lifecycle patterns to resources
- When to use resources vs existing patterns

### Learning Materials

**Blog Posts and Tutorials**
- "Introducing Resources: A New Reactive Primitive"
- "Converting from Class Helpers to Resources"
- "Building Reusable Data Loading with Resources"
- "Testing Resources in Isolation"

**Interactive Tutorials**
- Ember Tutorial additions demonstrating resource usage
- Playground examples for common patterns
- Step-by-step conversion guides

### Integration with Existing Learning

Resources complement existing Ember concepts rather than replacing them:

- **Components**: Still the primary UI abstraction, now with better tools for managing stateful logic
- **Services**: Still used for app-wide shared state, resources handle localized stateful processes
- **Helpers**: Resources can serve as stateful helpers, while pure functions remain as regular helpers
- **Modifiers**: Resources can encapsulate modifier-like behavior with better composition

## Drawbacks

### Learning Curve

**Additional Abstraction**: Resources introduce a new primitive that developers must learn. While they simplify many patterns, they add to the initial cognitive load for new Ember developers.

**Multiple Ways to Do Things**: Resources provide another way to manage state and side effects, potentially creating confusion about when to use components vs. services vs. resources.

### Performance Considerations

**Memory Overhead**: Each resource instance maintains its own cache and cleanup tracking, which could increase memory usage in applications with many resource instances.

**Re-computation**: Resources re-run their entire function when tracked dependencies change, which could be less efficient than more granular update patterns.

### Migration and Ecosystem Impact

**Existing Patterns**: The addition of resources doesn't make existing patterns invalid, but it might create inconsistency in codebases during transition periods.

**Addon Ecosystem**: Addons will need time to adopt resource patterns, potentially creating a split between "old" and "new" style addons.

**Testing Infrastructure**: Existing testing patterns and helpers may not work optimally with resources, requiring new testing utilities and patterns.

### TypeScript Complexity

**Advanced Types**: The resource type system, especially around composition with `use()`, involves complex TypeScript patterns that may be difficult for some developers to understand or extend.

**Decorator Limitations**: The `@use` decorator may not provide optimal TypeScript inference in all scenarios due to limitations in decorator typing.

## Alternatives

### Class-Based Resources

Instead of function-based resources, we could provide a class-based API similar to the current ember-resources library:

```js
class TimerResource extends Resource {
  @tracked time = new Date();
  
  setup() {
    this.timer = setInterval(() => {
      this.time = new Date();
    }, 1000);
  }
  
  teardown() {
    clearInterval(this.timer);
  }
}
```

**Pros**: More familiar to developers used to class-based patterns
**Cons**: More verbose, harder to compose, doesn't leverage functional programming benefits

### Built-in Helpers with Lifecycle

Extend the existing helper system to support lifecycle hooks:

```js
export default class TimerHelper extends Helper {
  @tracked time = new Date();
  
  didBecomeActive() {
    this.timer = setInterval(() => {
      this.time = new Date();
    }, 1000);
  }
  
  willDestroy() {
    clearInterval(this.timer);
  }
  
  compute() {
    return this.time;
  }
}
```

**Pros**: Builds on existing helper system
**Cons**: Limited to helper use cases, doesn't solve composition or broader lifecycle management

### Enhanced Modifiers

Expand the modifier system to handle more use cases:

```js
export default class DataModifier extends Modifier {
  modify(element, [url]) {
    // Fetch data and update element
  }
}
```

**Pros**: Builds on existing modifier system
**Cons**: Still limited to DOM-centric use cases, doesn't solve general stateful logic management

### Service-Based Solutions

Encourage using services for all stateful logic:

```js
export default class TimerService extends Service {
  @tracked time = new Date();
  
  startTimer() {
    this.timer = setInterval(() => {
      this.time = new Date();
    }, 1000);
  }
  
  stopTimer() {
    clearInterval(this.timer);
  }
}
```

**Pros**: Uses existing patterns
**Cons**: Too heavyweight for localized state, poor cleanup semantics, singleton limitations

### No Action (Status Quo)

Continue with current patterns and encourage better use of existing primitives.

**Pros**: No additional complexity or learning curve
**Cons**: Doesn't solve the identified problems, perpetuates scattered lifecycle management

## Unresolved questions

### Future Synchronization API

While this RFC focuses on the core resource primitive, a future `on.sync` API (similar to what Starbeam provides) could enable even more powerful reactive patterns. However, this is explicitly out of scope for this RFC to keep the initial implementation focused and stable.

### Integration with Strict Mode and Template Imports

How should resources work with Ember's strict mode and template imports? Should resources be importable in templates directly, or only through helper invocation?

### Performance Optimization Opportunities

Are there opportunities to optimize resource re-computation through more granular dependency tracking or memoization strategies?

### Testing Utilities

What additional testing utilities should be provided in the box for resource testing? Should there be special test helpers for resource lifecycle management?

### Debugging and Ember Inspector Integration

How should resources appear in Ember Inspector? What debugging information should be available for resource instances and their lifecycles?

### Migration Tooling

Should there be automated codemods to help migrate from existing patterns (class helpers, certain component patterns) to resources?

### Bundle Size Impact

What is the impact on bundle size for applications that don't use resources? Can the implementation be designed to be tree-shakeable?
