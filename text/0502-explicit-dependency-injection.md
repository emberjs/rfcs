---
stage: accepted
start-date: 2019-06-14T00:00:00.000Z # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions: 
teams:
  - data
  - framework
  - learning
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/502
project-link:
suite: 
---

# Explicit Service Injection

## Summary

This RFC adds overloads to the `service` decorator (formerly `inject`). They give better editor support and more ways to manage state tied to the lifetime of an application.

## Motivation

The main goal is to _use the platform_ and enable "go to definition" support from service definitions so developers can more easily discover where and how their service is defined. Additionally, addons need to create services that are not forced into the initial JS bundle of an application. If an addon is used in a split route, its services belong to that route too.

## Detailed design

The existing syntax, which this RFC does not deprecate:

```ts
@service notifications;
@service('notifications') notifications;
@service('messages/dispatcher') dispatcher;
```

has this behavior:
- service is not instantiated until accessed (even though the code for it is loaded)
- service is matched to a _file path_ within the `app/services` directory via custom build tooling
- usage with typescript requires that the developer import from the service file to `@service declare notifications: Notifications` - which is very repetitive
- service can be overridden later (if it has not previously been accessed) via `owner.register(key, factory)`
- service can be resolved later without a decorator via `owner.lookup(key)`


This RFC proposes a new default, retaining the benefits of the above described features without the downsides:

```ts
@service(Notifications) notifications;
@service(MessageDispatcher) dispatcher;

// or
notifications = service(this, Notifications);
dispatcher = service(this, MessageDispatcher);
```

This means that the shorthand syntax of using `@service` without a parameter should be discouraged, and the use of the service class should be used in its place.

Two exports from `@ember/service` replace the owner methods for class keys:

```ts
import { service, overrideService } from '@ember/service';

// lookup: the first argument is an owner, or an object that has an owner
const notifications = service(owner, Notifications);

// override: before the first lookup of the key on this owner
overrideService(owner, Notifications, TestNotifications);
```

`owner.lookup`, `owner.register`, and the application instance do not change. They keep their string-key behavior, and they do not accept class keys.

A string keeps today's behavior. A class key resolves to one instance per owner, and needs no registration step.

### what can be a service

A class-keyed service has two parts: the key, and the implementation that the owner instantiates for that key. Without an override, the key is also the implementation.

Keys and implementations follow the same protocol. `service` checks a value in this order:

1. If it has a `create` method, it is a factory, and `create` makes the instance. This covers `EmberObject` and `Service` subclasses, a plain object such as `{ create }`, and a function with a static `create`. It is the same factory protocol that `owner.register('service:x', factory)` accepts today.
2. Otherwise, if it is a class, the instance is made with `new`, and the constructor receives the owner. The class can be abstract, because a key can be only a contract.
3. Otherwise, if it is an arrow function, it is a thunk (see "overloading service in js/ts"), and its return value is the key.

In each case, the instance has the owner.

Any other value is not a key. Strings stay in the string namespace. Symbols and plain objects without `create` are not keys.


An override must be the key, or inherit from the key; see "the hierarchy check".


Libraries using services they are not the owners of are not solved by this RFC; see "out of scope: interface / shape matching". Such a library keeps using a string key.

### lookup forms

#### decorators

##### by string 

the existing behavior

```js
class Demo {
    @service('service') myService!: Service;
    @service() service!: Service;
    @service service!: Service;
}
```

##### by class

proposed by this RFC

- retains lazy instantiation 

```ts
class Demo {
    @service(Service) myService!: Service;
}
```

##### via function returning class

proposed by this RFC

- retains lazy instantiation 
- retains lazy definition / cycle-solving benefits 

```ts
class Demo {
    @service(() => Service) myService!: Service;
}
```

#### without decorators

proposed by this RFC

- retains TypeScript types ahead of Decorators being shipped
- retains lazy definition / cycle-solving benefits 
- can be '#privateField'
- can be used in getter, function, etc

```ts
class Demo {
    myService = service(this, Service);
}
```

**This form does not retain lazy instantiation.** A field initializer must evaluate to something, so deferring means returning a stand-in. The only candidate (which is not being proposed) is a `Proxy`, which:

- throws on private fields: `this.#state` is a `TypeError` when the receiver is a Proxy, and `#privateField` is listed above as a benefit of this form
- breaks identity: `this.myService === service(owner, Service)` is `false`
- shows as a Proxy in the debugger

A `tracked()` read as `this.myService.value` is lazy and correct, but makes the return value more verbose at every call site. The lazy non-decorator form is a getter:

```ts
class Demo {
    // resolves during construction
    eager = service(this, Service);                
    // resolves on first access.
    // repeat service() calls are idempotent.
    get lazy() { return service(this, Service); }  
}
```

#### overloading service in js/ts 

One export serves every form:

| call | meaning |
| --- | --- |
| `service('name')` | today's behavior, unchanged |
| `service(Key)` | lazy decorator |
| `service(() => Key)` | lazy decorator, cycle-tolerant |
| `service(this, Key)` | resolve now. `this` is an owner, or an object that has an owner. |

A thunk must be an arrow function, so that it can be told apart from a class.

### lookup by exact class

```ts
// no registration needed, because MyClass could be in a dynamic bundle
service(appInstance, MyClass);
```


Examples: 

 - **service override**
    ```ts
    class MyFooService {
      @tracked foo = 0;

      add() {
        this.foo++;
      }
    }
    ```
    Now, where this _IS_ Dependency Injection, and how we aren't just using the concrete class all the time is where you can do things like this

    ```ts
    class MyFooOverrideService extends MyFooService {
      add() {
        this.foo += 2;
      }
    }

    // both stubbing (in a test), or clobbering, would look the same
    overrideService(appInstance, MyFooService, MyFooOverrideService);

    const foo = service(appInstance, MyFooService);

    foo instanceof MyFooOverrideService // true
    foo instanceof MyFooService // true
    ```

 - **no registration**
    ```ts
    class MyFooService {
      @tracked foo = 0;

      add() {
        this.foo++;
      }
    }

    // the above service has no registration and no override
    const myFooService = service(appInstance, MyFooService);

    myFooService instanceof MyFooService // true
    ```

 - **typescript fastboot example with instance initializers**
  
    `app/services/cookies/cookie-service.ts`:

    ```ts
    export abstract class CookieService {
      abstract getValue(key: string): string | undefined;

      abstract setValue(key: string, value: string): void;
    }
    ```

    `app/components/my-component.js`:

    ```ts
    import Component from '@glimmer/component';
    import { service } from '@ember/service';
    import { CookieService } from 'app-name/services/cookies/cookie-service';

    class MyComponent extends Component {
      @service(CookieService) cookie;
    }
    ```

    `app/services/cookies/fastboot-service.ts`:

    ```ts
    import { CookieService } from 'app-name/services/cookies/cookie-service';

    export class FastbootCookieService extends CookieService {
      getValue(key: string) {
        return undefined;
      }
      // ...
    }
    ```

    `app/services/cookies/browser-service.ts`:

    ```ts
    import { CookieService } from 'app-name/services/cookies/cookie-service';

    export class BrowserCookieService extends CookieService {
      getValue(key: string) {
        return document.cookie
          .split('; ')
          .find((pair) => pair.startsWith(`${key}=`))
          ?.split('=')[1];
      }
      // ...
    }
    ```

    `app/instance-initializers/register-cookie-service.ts`:

    ```ts
    import { overrideService } from '@ember/service';
    import { CookieService } from 'app-name/services/cookies/cookie-service';
    import { FastbootCookieService } from 'app-name/services/cookies/fastboot-service';
    import { BrowserCookieService } from 'app-name/services/cookies/browser-service';

    export function initialize(appInstance) {
      const fastboot = appInstance.lookup('service:fastboot');

      if (fastboot?.isFastBoot) {
        overrideService(appInstance, CookieService, FastbootCookieService);
      } else {
        overrideService(appInstance, CookieService, BrowserCookieService);
      }
    }

    export default { initialize };
    ```

    A key can be a pure contract: an abstract class whose abstract members leave nothing at runtime.

#### the hierarchy check

`overrideService` makes sure that the override either is the key or inherits from the key. This will prevent the ability to register unrelated classes that would break the implied class hierarchy that is assumed with dependency injection. 

```ts
overrideService(appInstance, CookieService, SomethingUnrelated);
// Error: SomethingUnrelated is neither CookieService nor a subclass of it.
```

A subclass inherits from its class. A factory object inherits from a key through its prototype chain, for example `{ __proto__: key, create() {} }`.

Note that this makes inheritance the only way to substitute an implementation, which is the constraint an interface-matching followup would relax.

#### overriding after a key has been resolved

Today, `owner.register` after a lookup silently does nothing: the container has cached the instance and never consults the new registration. The usual way to hit it is stubbing after `render`, and the result is a test that passes for the wrong reason. `overrideService` asserts instead:

```ts
service(owner, Counter);
overrideService(owner, Counter, StubCounter);
// Error: Counter has already been resolved on this owner. Override before the
// first lookup -- in a test, before render or visit.
```

#### `hasRegistration` does not change

`hasRegistration('service:foo')` answers "can the resolver find this name", and the answer can be no. For a class key the answer is always yes, because the key can instantiate itself. Thus `hasRegistration` stays string-only.

### services no longer need a base class

There is no name to resolve and no `create` contract to satisfy, so a service can be a plain class:

```ts
export default class Counter {
  @tracked count = 0;

  constructor() {
    registerDestructor(this, () => this.cleanup());
  }

  increment = () => this.count++;

  cleanup() {
    // release anything this service holds
  }
}
```

Extending `Service` stays supported; it stops being mandatory. A service extends it when its fields need the owner.

#### the owner is not available in a plain class's field initializers

The owner is attached after the constructor returns:

```ts
class Broken {
  log = service(this, Logger);  // Error: `this` has no owner yet
}

class Works {
  #logger: Logger;

  constructor(owner: Owner) {
    this.#logger = service(owner, Logger);   // inject from the parameter
  }

  get other() { return service(this, Other); }  // or resolve after construction
}
```

A subclass of `Service` does not hit this. The constructor of `Service` sets the owner, and JavaScript runs the fields of a subclass after `super()` returns:

```ts
import Service, { service } from '@ember/service';

class AlsoWorks extends Service {
  log = service(this, Logger);  // the owner is already set
}
```

### out of scope: interface / shape matching

- javascript does not have interfaces

A library cannot name a key the *app* owns, which is [ember-polaris-service#19](https://github.com/chancancode/ember-polaris-service/issues/19). The library declares a dependency, the app supplies the implementation, and there is no class both sides can import.

> [!NOTE]
> The ember-intl case still works with class keys. A library that depends on ember-intl uses its class as the key. The app subclasses that class, as the hierarchy check requires, and overrides it with `overrideService(appInstance, IntlService, AppIntlService)` in an instance initializer, before the first lookup of `IntlService`. Lookup does not search for subclasses, and an override after the first lookup asserts; see "overriding after a key has been resolved".

Matching a key structurally -- "any registered class with these members" -- would close that gap. Class keys do not need it, and it has its own hazards:

- renaming a method silently unbinds a provider
- two candidates matching one key has no defensible answer
- `abstract` members leave nothing at runtime to match against

**Use a string key for this case.** `@service('transport')` keeps working as it does now, so nothing is lost relative to the status quo. A library whose dependency is app-supplied keeps the string, and converts if and when a followup RFC gives it something better.

One constraint that does belong here, because it constrains the implementation:
class lookup must not fall back to *name* resolution. `service(owner, FeatureFlags)` must not resolve `app/services/feature-flags.ts` because the dasherized name happens to match, or "go to definition" stops being trustworthy -- which is the motivation for the RFC. Class keys and string names are separate namespaces that coexist.

### module cycles

ESM permits the cycle; what fails is reading a binding before the owning module has finished evaluating, and a decorator *argument* is evaluated at class-definition time.

`app/domain/cycle/editor.ts`:

```ts
import History from './history.ts';

export default class Editor {
  @service(() => History) declare history: History;
}
```

`app/domain/cycle/history.ts`:

```ts
import Editor from './editor.ts';

export default class History {
  @service(() => Editor) declare editor: Editor;
}
```

**Both** sides need the thunk. Whichever module is imported first works with the direct form, but which one that is depends on the app's import order rather than on either file. The thunk is the default for mutually-dependent services, not an advanced escape hatch.

### TypeScript

`lookup`'s return type is derived from the `Registry` declaration-merging registry, keyed on the string name, which is why every service in a typed app carries this:

```ts
declare module '@ember/service' {
  interface Registry {
    'feature-flags': FeatureFlags;
  }
}
```

With a class key the instance type comes from the class, so the declaration merging disappears. The non-decorator form also drops the `@service declare foo: Foo` duplication described above, because TypeScript infers the field type from the return value. A decorator cannot change the type of the field it decorates, so the decorator form keeps the annotation:

```ts
flags = service(this, FeatureFlags);                     // inferred as FeatureFlags
@service(FeatureFlags) declare otherFlags: FeatureFlags;  // annotation still required
```

`Key` must allow abstract constructors, because a key that is only a contract is the most useful kind of key:

```ts
interface Factory<T> {
  create(props: object): T;
}

type Key<T> = (abstract new (owner: Owner) => T) | Factory<T>;

function service<T>(context: Owner | object, key: Key<T>): T;
function overrideService<T>(owner: Owner, key: Key<T>, impl: Key<T>): void;
```

The public `Owner` type does not change.

### Usage in testing

#### Acceptance Tests

Given that we have a service:
```ts
// app-name/services/my-service
export default class MyService {
  someMethodThatIsInTheSuperClass() {
    return 'not stubbed';
  }
}
```

And assuming that in a route there is a service injection like the following:
```ts
import Route from '@ember/routing/route';
import { service } from '@ember/service';
import ServiceToOverride from 'app-name/services/my-service';

export default class TheRoute extends Route {
  @service(ServiceToOverride) myService;
  // ...
}
```

During an acceptance test, it can be overridden by doing the following:
```ts
import { module } from 'qunit';
import { setupApplicationTest } from 'ember-qunit';
import { overrideService } from '@ember/service';

import ServiceToOverride from 'app-name/services/my-service';

module('Acceptance | test a thing', function(hooks) {
  setupApplicationTest(hooks);

  hooks.beforeEach(function() {
    class MockService extends ServiceToOverride {
      someMethodThatIsInTheSuperClass() {
        return 'stubbed';
      }
    }

    // override under the same 'key', ServiceToOverride
    overrideService(this.owner, ServiceToOverride, MockService);
  });
});
```

The override under the key is the whole of it. There is no second `inject` step, because the injection resolves through the owner on access rather than being assigned at instantiation. The override does have to happen before `visit`; see "overriding after a key has been resolved".

#### Integration Tests

```ts
import { module } from 'qunit';
import { setupRenderingTest } from 'ember-qunit';
import { render } from '@ember/test-helpers';
import { overrideService } from '@ember/service';

import LocationService from 'app-name/services/location';

class LocationStub extends LocationService {
  getCurrentCity() {
    return 'Indianapolis';
  }

  getCurrentCountry() {
    return '???';
  }
}

module('Integration | Component | location-indicator', function(hooks) {
  setupRenderingTest(hooks);

  hooks.beforeEach(function() {
    overrideService(this.owner, LocationService, LocationStub);
  });
});
```

The stub does not have to live anywhere in particular or be reachable by the resolver -- it is a class in the test file. There is no name to get right and no `app/services/` file to shadow.

#### Unit tests

A service with no owner-dependent behavior needs no setup:

```ts
test('it counts', function (assert) {
  const counter = new Counter();

  counter.increment();

  assert.strictEqual(counter.count, 1);
});
```

Not available today, since `Service` has to be created through the container.

## How we teach this

Instead of using strings or inferred injections, the guides should be updated to use the Class definition of a service that it intends to inject.

The guides already cover usage of services, as well as techniques for setting up [any class for injection](https://guides.emberjs.com/release/in-depth-topics/native-classes-in-depth/#toc_using-injection) (under "in depth topics").

The "Application State" guide in [guides-source#2237](https://github.com/ember-learn/guides-source/pull/2237) teaches state tied to the lifetime of the owner. Class-keyed services can use that guide as their starting point.

Migration instructions from string keys to class keys can come with a future RFC that deprecates string-based services, once a compat library proxies the string-instantiated service to the real one.

Things the guides must cover, because each is a place reasonable code fails:

- We should default to lazily-initialized forms
- `@service(Key)` is the default form (shortest, nicest for javascript). 
- plain-class services cannot inject in field initializers -- teach the getter
- override before first lookup, which in a test means before `render`/`visit`. The testing guides do not cover this yet.
- a dependency the app supplies, rather than the library, still uses a string key

The service directory only matters for services still reached by a string key. A class-keyed service can live next to its consumers, since the import is the registration.

## Drawbacks

- more verbose
- disruptive for libraries providing services
- the non-decorator form loses lazy instantiation. A getter gets it back with no new API. A future RFC can add a `lazy()` utility, for example `prop = lazy(service(this, Key))`, which wraps the service in a proxy. That utility is not part of this RFC.
- class keys create module edges string keys did not, so cycles become possible where they could not exist before. The thunk is easy to forget, and forgetting it fails at boot.
- class names are minified in production, so the container cache and the Ember Inspector lose the readable names string keys give them
- both styles coexist 

## Alternatives

 - in C# + asp.net core, dependency injections are resolved in the constructor of a class. This would be _even more verbose_ than what is being proposed in this RFC, as for many situations in Ember, the constructor can be omitted. 

 - just use `<library>`
   - many existing libraries for other ecosystems: very verbose, and use class-decorators, and/or constructor decorators -- which is more like C# + asp.net core.

   - some existing explorations
     - https://github.com/chancancode/ember-polaris-service
     - https://ember-primitives.pages.dev/6-utils/createService

   a goal of the RFC is to signal to the community which approach will be supported and is safest. ember-polaris-service did a lot of the exploration early. ember-primitives' createService demonstrates that, if you really want, you don't need a service abstraction at all (leaning in to "don't mock", and "WeakMap on the owner is good")

 - symbol tokens instead of classes: `service(owner, TRANSPORT)`. Solves the same coupling problem and avoids module cycles entirely, since a symbol has no dependencies. Loses the thing the RFC is named for -- a symbol is not ctrl+clickable to an implementation, only to its own declaration.

 - shape matching in this RFC rather than a followup. Solves polaris-service#19 now, at the cost of a much larger surface and a fuzzier one; see "out of scope".

 ## Unresolved Questions

 - Does `@ember/service` export a way to ask whether a key has an override, or whether a key was already resolved on an owner?

 - Should looking up an abstract key with no override throw? At runtime it builds an instance of the abstract class, which has none of the abstract members.

 - What does the Ember Inspector show for a class-keyed instance in a production build, where `Klass.name` is minified?

 - Does the `@service` decorator need to keep supporting assignment (`this.myService = stub`)?

## Appendix

- Implementation

    This RFC does not prescribe how class-keyed instances are stored. One approach keeps them in a `WeakMap` keyed on the owner, as [`createService`](https://ember-primitives.pages.dev/6-utils/createService) does. That approach leaves `Registry`, the container, and the string path untouched.

- Where this has been validated

    [ember-rfc-502-explicit-services-polyfill](https://github.com/NullVoxPopuli/ember-rfc-502-explicit-services-polyfill) implements this RFC for ember-source 3.28 and newer. It is tested in an ember-cli Vite app, a classic broccoli app, an ember.nvp app, and a broccoli app on ember-source 3.28.
