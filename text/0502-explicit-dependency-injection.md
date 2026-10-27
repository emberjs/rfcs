---
stage: accepted
start-date: 2019-06-14T00:00:00.000Z # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions: 
teams: # delete teams that aren't relevant
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

When learning Ember and/or Dependency Injection, the question comes about how magic the injection strings are. The goal of this RFC is to propose an alternate default for injections that allow ctrl+clickability / "go to definition" from the injection to the class that defines the type of the injected instance.

Note: while this RFC mainly talks in terms in services, this applies to all injections.

## Motivation

The main goal is to _use the platform_ and enable "go to definition" support from service definitions so developers can more easily discover the where and how their service is defined. Additionally, addons need to be able to create services that are not forced to be inluded in the initial JS bundle of applications -- if an addon is used in a split route, so, too, should their services.

## Detailed design

The existing, not deprecated as a part of this RFC,  
```ts
@service notifications;
@service('notifications') notifications;
@service('messages/dispatcher') dispatcher;
```

has this behavior:
- service is not instantiated until accessed (even though the code for it is loaded)
- service is matched to a _file path_ within the `app/services` directory via custom build tooling
- usage with typescript requires that the developer import from the service file to `@service declare notifications: Notifications` - which is very repetitive
- service can be overridden later (if they have not previously been accessed) via `owner.register(key)`
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

At present, the service decorator wraps around the `ApplicationInstance#lookup` method -- something like this (roughly / hand-waiving the implementation details of decorators and getting access to the app instance):

```ts
function service(name) {
  return appInstance.lookup(`service:${name}`);
}
```

Instead, with this RFC, the service pseudo-function should check for the type of the parameter, and:

```ts
function service(nameOrClass) {
  if (typeof nameOrClass === 'string') {
    return appInstance.lookup(`service:${name}`);
  }

  return appInstance.lookup(nameOrClass);
}
```

In order for `lookup` to be able to take a class definition as an argument, there will need to be an alternative way to _lookup_ instances of services by the class. Though, when a class is used for lookup, if there is no existing registration found, lookup _will register the class and instantiate it for you_.


Libraries using services they are not the owners of are not solved by this RFC; see
"out of scope: interface / shape matching". Such a library keeps using a string key,
which is what it does today.

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

**This form does not retain lazy instantiation.** A field initializer must evaluate
to something, so deferring means returning a stand-in. The only candidate (which is not being proposed) is a
`Proxy`, which:

- throws on private fields: `this.#state` is a `TypeError` when the receiver is a
  Proxy, and `#privateField` is listed above as a benefit of this form
- breaks identity: `this.myService === owner.lookup(Service)` is `false`
- shows as a Proxy in the debugger

A `tracked()` read as `this.myService.value` is lazy and correct, but makes the return value more verbose to use, which would make the following not work 

site. The lazy non-decorator form is a getter:

```ts
class Demo {
    // resolves during construction
    eager = service(this, Service);                
    // resolves on first access.
    // repeat service() calls are idempontent.
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
| `service(this, Key)` | resolve now |

### lookup by exact class

Consider:

```ts
appInstance.register(MyClass, MyClass);
appInstance.lookup(MyClass);
```

is the same as 
```ts
// no registration needed, because MyClass could be in a dynamic bundle
appInstance.lookup(MyClass);
```

This will be most handy for decorators or other common abstractions that desire to interact with services (such as the router or store), but have direct access to them from the component or route context.

Examples: 

 - **service registration and override**
    ```ts
    class MyFooService {
      @tracked foo = 0;

      add() {
        this.foo++;
      }
    }

    appInstance.register(MyFooService, MyFooService);
    ```
    Now, where this _IS_ Dependency Injection, and how we aren't just using the concrete class all the time is where you can do things like this

    ```ts
    class MyFooOverrideService extends MyFooService {
      add() {
        this.foo += 2;
      }
    }

    // both stubbing (in a test), or clobbering, would look the same
    appInstance.register(MyFooService, MyFooOverrideService);

    const service = appInstance.lookup(MyFooService);

    service instanceof MyFooOverrideService // true
    service instanceof MyFooService // true
    ```

 - **less typing for lazy registration**
    ```ts
    class MyFooService {
      @tracked foo = 0;

      add() {
        this.foo++;
      }
    }

    // the above service has not been registered yet
    const myFooService = appInstance.lookup(MyFooService);
    // first time lookup without registration will register for you.

    myFooService instanceof MyFooService // true
    ```

 - **typescript fastboot example with instance initializers**
  
    ```ts
    // app/services/cookies/cookie-service.ts
    export abstract class CookieService {
      abstract getValue(key: string): string | undefined;

      abstract setValue(key: string, value: string): void;
    }

    // app/components/my-component.js
    import Component from '@glimmer/component';
    import { service } from '@ember/service';

    class MyComponent extends Component {
      @service(CookieService) cookie;
    }

    // app/services/cookies/fastboot-service.ts
    import { CookieService } from 'app-name/services/cookies/cookie-service';

    export class FastbootCookieService extends CookieService {
      getValue(key: string) {
        return undefined;
      }
      // ...
    }

    // app/services/cookies/browser-service.ts
    import { CookieService } from 'app-name/services/cookies/cookie-service';

    export class BrowserCookieService extends CookieService {
      getValue(key: string) {
        return browser.cookies.get({
          name: key,
          url: window.location.href
        })
      }
      // ...
    }

    // app/instance-initializers/register-cookie-service.ts
    import { CookieService } from 'app-name/services/cookies/cookie-service';
    import { FastbootCookieService } from 'app-name/services/cookies/fastboot-service';
    import { BrowserCookieService } from 'app-name/services/cookies/browser-service';

    export function initialize(appInstance) {
      const fastboot = appInstance.lookup('service:fastboot');

      if (fastboot?.isFastBoot) {
        appInstance.register(CookieService, FastbootCookieService);
      } else {
        appInstance.register(CookieService, BrowserCookieService);
      }
    }

    export default { initialize };
    ```

    A key can be a pure contract with no runtime footprint at all.

#### the hierarchy check

Logic will be added to the register method to ensure that the lookup type either is the same as the service instance's type or is an ancestor type. This will prevent the ability to register unrelated classes that would break the implied class hierarchy that is assumed with dependency injection. 

```ts
appInstance.register(CookieService, SomethingUnrelated);
// Error: SomethingUnrelated is neither CookieService nor a subclass of it.
```

Note that this makes subclassing the only way to substitute an implementation, which
is the constraint an interface-matching followup would relax.

#### registering after a key has been resolved

Today this silently does nothing: the container has cached the instance and never
consults the new registration. The usual way to hit it is stubbing after `render`,
and the result is a test that passes for the wrong reason. Class-keyed registration
should assert instead:

```ts
lookup(owner, Counter);
register(owner, Counter, StubCounter);
// Error: Counter has already been resolved on this owner. Register before the
// first lookup -- in a test, before render or visit.
```

#### `hasRegistration` does not survive the move to class keys

`hasRegistration('service:foo')` answers "can the resolver find this name", and the
answer can be no. For a class key it is trivially yes for every key -- worst case the
key instantiates itself -- so the method becomes useless rather than different.

Two separate predicates replace it:

- is an explicit override bound for this key on this owner?
- has this key been resolved on this owner yet?

The second is what the assertion above needs.

### services no longer need a base class

There is no name to resolve and no `create` contract to satisfy, so a service can be
a plain class:

```ts
export default class Counter {
  @tracked count = 0;

  constructor(owner: Owner) {
    registerDestructor(this, () => this.cleanup());
  }

  increment = () => this.count++;
}
```

`InternalFactoryManager#create` currently asserts
`typeof this.class.create === 'function'`, so only `EmberObject` descendants can be
registered (or those simulating the create method from `EmberObject`). Class-keyed lookup needs both paths:

- static `create` (an `EmberObject` descendant): instantiate through it, with the
  owner set on the props object, as today
- otherwise: `new Impl(owner)`, then `setOwner(instance, owner)`

Extending `Service` stays supported; it stops being mandatory.

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

`EmberObject`-based services do not hit this, because `create(props)` sets the owner
before `init` runs.

### out of scope: interface / shape matching

- javascript does not have interfaces

A library cannot name a key the *app* owns, which is
[ember-polaris-service#19](https://github.com/chancancode/ember-polaris-service/issues/19):
the library declares a dependency, the app supplies the implementation, and there is
no class both sides can import.

> [!NOTE]
> For the ember-intl case, where a library _does_ depend on ember-intl, and uses their class key, since the app version of the intl service would _extend_ (and be required to extend) the intl service, because we have the hierarchy lookup, this use case still works with explicit service references. An important caveat is that the first access registers the instance -- so upon app boot, it should interact with the intl service, such as setting the default locale or reading the locale from localStorage / the URL. 

Matching a key structurally -- "any registered class with these members" -- would
close that gap, but it is a large feature with its own hazards (renaming a method
silently unbinds a provider; two candidates matching one key has no defensible
answer; `abstract` members leave nothing at runtime to match against), and it is not
needed to ship class keys.

**Use a string key for this case.** `@service('transport')` continues to work
exactly as it does today, so nothing is lost relative to the status quo -- this RFC
adds a second way to name a dependency rather than replacing the first. A library
whose dependency is app-supplied keeps the string, and converts if and when a
followup RFC gives it something better.

One constraint that does belong here, because it constrains the implementation:
class lookup must not fall back to *name* resolution. `lookup(FeatureFlags)` must not
resolve `app/services/feature-flags.ts` because the dasherized name happens to match,
or "go to definition" stops being trustworthy -- which is the motivation for the RFC.
Class keys and string names are separate namespaces that coexist.

### module cycles

ESM permits the cycle; what fails is reading a binding before the owning module has
finished evaluating, and a decorator *argument* is evaluated at class-definition time.

```ts
// app/domain/cycle/editor.ts
import History from './history.ts';

export default class Editor {
  @service(() => History) declare history: History;
}

// app/domain/cycle/history.ts
import Editor from './editor.ts';

export default class History {
  @service(() => Editor) declare editor: Editor;
}
```

**Both** sides need the thunk. Whichever module is imported first works with the
direct form, but which one that is depends on the app's import order rather than on
either file. The thunk is the default for mutually-dependent services, not an
advanced escape hatch.

### TypeScript

`lookup`'s return type is derived from the `DIRegistry` declaration-merging registry,
keyed on the string name, which is why every service in a typed app carries this:

```ts
declare module '@ember/service' {
  interface Registry {
    'feature-flags': FeatureFlags;
  }
}
```

With a class key the instance type comes from the class, so the declaration merging
disappears, as does the `@service declare foo: Foo` duplication from Motivation:

```ts
@service(FeatureFlags) declare flags: FeatureFlags;  // annotation now redundant
```

Two signature notes:

- `Key` must allow abstract constructors (`abstract new (owner: Owner) => T`); a
  token that is only a contract is the most useful kind of key
- `hasRegistration` and `unregister` are not on the public `Owner` type today, they
  live on the private `RegistryProxy`. The testing examples use them, so this RFC
  needs to say whether they become public.

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
import ServiceToOverride from 'app-name/services/my-service';

export default class TheRoute extends Route {
  @service(ServiceToOverride) myService;
  // ...
}
```

During an acceptance test, it can be overwritten by doing the following:
```ts
import { module, test } from 'qunit';
import { setupApplicationTest } from 'ember-qunit';

import ServiceToOverride from 'app-name/services/my-service';

module('Acceptance | test a thing', function(hooks) {
  setupApplicationTest(hooks);

  hooks.beforeEach(function() {
    class MockService extends ServiceToOverride {
      someMethodThatIsInTheSuperClass() {
        return 'stubbed';
      }
    }

    // register under the same 'key', ServiceToOverride
    this.owner.register(ServiceToOverride, MockService);
  });
});
```

Registering under the key is the whole of it. There is no second `inject` step,
because the injection resolves through the owner on access rather than being assigned
at instantiation. The registration does have to happen before `visit`; see
"registering after a key has been resolved".

### Integration Tests

```ts
import { module, test } from 'qunit';
import { setupRenderingTest } from 'ember-qunit';
import { render } from '@ember/test-helpers';

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
    this.owner.register(LocationService, LocationStub);
  });
});
```

The stub does not have to live anywhere in particular or be reachable by the
resolver -- it is a class in the test file. There is no name to get right and no
`app/services/` file to shadow.

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

The guides already cover usage of services, as well as techniques for setting up [any class for injection](https://guides.emberjs.com/release/in-depth-topics/native-classes-in-depth/#toc_using-injection) (albeit under "in depth topics") -- some of this probably could be expanded on but would be out of scope for this RFC. 

For libraries and apps wanting to incrementally migrate from string to explicit, we can provide instructions during a future depreaction-of-string-based-services RFC after a compat library exists, (proxying the string-instantiated service to the real one).

Things the guides must cover, because each is a place reasonable code fails:

- We should default to lazily-initialized forms
- `@service(Key)` is the default form (shortest, nicest for javascript). 
- plain-class services cannot inject in field initializers -- teach the getter
- register before first lookup, which in a test means before `render`/`visit`
  This doesn't exist today in the guides, as our guides on testing could use some work
- a dependency the app supplies, rather than the library, still uses a string key

The service directory only matters for services still reached by a string key. A
class-keyed service can live next to its consumers, since the import is the
registration.

## Drawbacks

- more verbose
- disruptive for libraries providing services
- the non-decorator form loses lazy instantiation 
- class keys create module edges string keys did not, so cycles become possible where
  they could not exist before. The thunk is a workaround, easy to forget, and
  forgetting it fails at boot.
- class names are minified in production, so the container cache and the Ember
  Inspector lose the readable names string keys give them
- both styles coexist 

## Alternatives

 - in C# + asp.net core, dependency injections are resolved in the constructor of a class. This would be _even more verbose_ than what is being proposed in this RFC, as for many situations in Ember, the constructor can be omitted. 

 - just use `<library>`
   - many existing libraries for other ecosystems: very verbose, and use class-decorators, and/or constructor decorators -- which is more like C# + asp.net core.

   - some existing explorations
     - https://github.com/chancancode/ember-polaris-service
     - https://ember-primitives.pages.dev/6-utils/createService 
   a goal of the RFC is to signal to the community which approach will be supported and is safest.
   ember-polaris-service did a lot of the exploration early, and ember-primitives' createService is a demonstration of how, if you really want, you don't need a service abstraction at all (leaning in to "don't mock", and "WeakMap on the owner is good")

 - symbol tokens instead of classes: `owner.lookup(TRANSPORT)`. Solves the same
   coupling problem and avoids module cycles entirely, since a symbol has no
   dependencies. Loses the thing the RFC is named for -- a symbol is not
   ctrl+clickable to an implementation, only to its own declaration.

 - shape matching in this RFC rather than a followup. Solves polaris-service#19 now,
   at the cost of a much larger surface and a fuzzier one; see "out of scope".

 ## Unresolved Questions

 - Should `hasRegistration` accept a class at all? It may be string-only forever,
   with two new predicates added alongside.

 - What does the Ember Inspector show for a class-keyed instance in a production
   build, where `Klass.name` is minified?

 - Does the `@service` decorator need to keep supporting assignment
   (`this.myService = stub`)?

## Appendix

- Previous Alternative: Convert the dependency injection lookup to a `WeakMap<Klass, instanceof Klass>`

    This would likely result in faster lookup, but would require more upfront work to change how the lookup method works on the `ApplicationInstance` / "owner". This _could_ be a separate RFC, but without benchmarking it's hard to say if this massive of a change would be worth it.

    Moved out of `Alternatives`, because this RFC should not explicitly recommend or unrecommend specific implementation styles. A `WeakMap` style of managing singletons tied to an owner is exactly the strategy that [`createService`](https://ember-primitives.pages.dev/6-utils/createService) from ember-primitives uses.

    **Update:** this turned out to be *less* upfront work rather than more, because
    it leaves `Registry` and the string path untouched instead of extending them. It
    is now the recommended direction rather than a rejected alternative; see
    "Implementation notes".

- Where this has been validated

    "Detailed design" has been implemented and tested against `ember-source` 7 in a
    playground app.

    The owner already *is* the container, so no parallel container is needed. The
    mechanism is a `WeakMap` keyed on the owner, using only `getOwner` and `setOwner`
    from `@ember/owner`. Instances are per-owner singletons because the outer key is
    the owner, and are destroyed with the owner via `associateDestroyableChild`.
    About 140 lines, including the decorator.



- A class key needs no name

    And also, no resolution, and no resolver, so everything class-keyed can live in a `WeakMap` keyed on the owner:

    ```ts
    const INSTANCES = new WeakMap<Owner, Map<Key, object>>();
    const BINDINGS = new WeakMap<Owner, Map<Key, Key>>();
    ```

    `Registry` and the string path are then untouched, which is also what keeps the two
    namespaces separate by construction rather than by convention. This is the same strategy
    [`createService`](https://ember-primitives.pages.dev/6-utils/createService) uses.

    What does need to change inside `container`:

    - `InternalFactoryManager#create` must not require a static `create`
    - destruction has to cover class-keyed instances. `destroyDestroyables` calls
      `value.destroy()` if it happens to exist, which is not enough for plain classes;
      `associateDestroyableChild(owner, instance)` at instantiation handles it, and makes
      `registerDestructor` work with no base class.

