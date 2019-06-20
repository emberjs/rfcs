- Start Date: 2019-06-14
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: https://github.com/emberjs/rfcs/pull/502
- Tracking: (leave this empty)

# Explicit Service Injection

## Summary

When learning Ember and/or Dependency Injection, the question comes about how magic the injection strings are. The goal of this RFC is to propose an alternate default for injections that allow ctrl+clickability / "go to definition" from the injection to the class that defines the type of the injected instance.

Note: while this RFC mainly talks in terms in services, this applies to all injections.

## Motivation

The main goal is to _use the platform_ and enable "go to definition" support from service definitions so developers can more easily discover the where and how their service is defined.

## Detailed design

instead of:
```ts
@service notifications;
@service('notifications') notifications;
@service('messages/dispatcher') dispatcher;
```
This will be the new default:
```ts
@service(NotificationsService) notifications;
@service(MessageDispatcherService) dispatcher;
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

Consider:

```ts
appInstance.register(MyClass, MyClass);
appInstance.lookup(MyClass);
```

is the same as 
```ts
// no registration
appInstance.lookup(MyClass);
```

This will be most handy for decorators or other common abstractions that desire to interact with services (such as the router or store), but have direct access to them from the component or route context.

On the `Registry`, there already exists a reference to the class definition when registering an entry to the container.

```ts
registry.register('model:user', Person, {singleton: false });
registry.register('fruit:favorite', Orange);
registry.register('communication:main', Email, {singleton: false});
```

A new map can exist on the registry that can appropriately be wired up to register, unregister, etc to handle the lookup-by-class-definition at the same time as the current registry behaves, allowing for backward compatibility with todays injection usage.

Examples: 

 - **service registration and override**
    ```ts
    class MyFooService extends Service {
      @tracked foo = 0;

      add() {
        this.foo++;
      }
    }

    appInstance.register(MyFooService, MyFooService);
    // this would register MyFooService on the *key* MyFooService,
    // just as today, it would look like
    appInstance.register('service:my-foo-service', MyFooService);
    ```
    Now, where this _IS_ Dependency Injection, and how we aren't just using the concrete class all the time is where you can do things like this

    ```ts
    // note that this must share ancestry with the registered service.
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
    class MyFooService extends Service {
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
    // app/services/cookie/cookie-service.ts
    abstract class CookieService {
      abstract getValue(key: string): string {}

      abstract setValue(key: string, value: string): void {}
    } 

    // app/components/my-component.js
    class MyComponent extends Component {
      @service(CookieService) cookie;
    }

    // app/services/cookie/fastboot-service.js
    import CookieService from 'app-name/services/cookies/cookie-service';

    class FastbootCookieService extends CookieService {
      // ...
      getValue(key: string) {
        return '';
      }
    }

    // app/services/cookie/browser-service.js
    import CookieService from 'app-name/services/cookies/cookie-service';

    class BrowserCookieService extends CookieService {
      getValue(key: string) {
        return browser.cookies.get({
          name: key,
          url: window.location.href
        })
      }
      /// ...
    }

    // app/instance-initializers/register-cookie-service;
    import CookieService from 'app-name/services/cookies/cookie-service';
    import FastbootCookieService from 'app-name/services/cookies/fastboot-service';
    import BrowserCookieService from 'app-name/services/cookies/browser-service';

    export function initialize(appInstance) {
      cost fastboot = appInstance.lookup('service:fastboot');
      
      if (fastboot.isFastBoot) {
        appInstance.register(CookieService, FastbootCookieService);
      } else {
        appInstance.register(CookieService, BrowserCookieService);
      }
    }

    export default { initialize };
    ```

Logic will be added to the register method to ensure that the lookup type either is the same as the service instance's type or is an ancestor type. This will prevent the ability to register unrelated classes that would break the implied class hierarchy that is assumed with dependency injection. 

### Usage in testing

#### Acceptance Tests

Given that we have a service:
```ts
// app-name/services/my-service
export default class MyService extends Service {
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
import { getContext } from '@ember/test-helpers';

import ServiceToOverride from 'app-name/services/my-service';

module('Acceptance | test a thing', function(hooks) {
  setupApplicationTest(hooks);

  hooks.beforeEach(function() {
    let { owner } = getContext();

    class MockService extends ServiceToOverride {
      someMethodThatIsInTheSuperClass() {
        return 'stubbed';
      }
    }

    // register under the same 'key', ServiceToOverride
    owner.register(ServiceToOverride, MockService);

    // re-inject on the-route
    owner.application.inject('route:the-route', 'myService', ServiceToOverride);
  });
});
```

### Integration Tests

```ts
import { module, test } from 'qunit';
import { setupRenderingTest } from 'ember-qunit';
import { render } from '@ember/test-helpers';
import hbs from 'htmlbars-inline-precompile';
import Service from '@ember/service';

import LocationService from 'app-name/services/location';

class LocationStub extends LocationService {
  getCurrentCity() {
    return 'Indianapolis';
  }

  getCurrentCountry() {
    return '???';
  }
});

module('Integration | Component | location-indicator', function(hooks) {
  setupRenderingTest(hooks);

  hooks.beforeEach(function(assert) {
    this.owner.register(LocationService, LocationStub);
  });
});
```


## How we teach this

Instead of using strings or inferred injections, the guides should be updated to use the Class definition of a service that it intends to inject.

Ember Inspector and the unstable-ember-language-server will likely need to be updated to support this kind of lookup.

## Drawbacks

- more verbose

## Alternatives

 - Convert the dependency injection lookup to a `WeakMap<Klass, instanceof Klass>`

    This would likely result in faster lookup, but would require more upfront work to change how the lookup method works on the `ApplicationInstance` / "owner". This _could_ be a separate RFC, but without benchmarking it's hard to say if this massive of a change would be worth it.

 - in C# + asp.net core, dependency injections are resolved in the constructor of a class. This would be _even more verbose_ than what is being proposed in this RFC, as for many situations in Ember, the constructor can be omitted. 


 ## Unresolved Questions

 These may be more for implementation, but as I was tracing how injections work.. it looks like the control flow path is:
  - `@service`
  - `appInstance.lookup`
    - `engineInstance.lookup` (Application inherits from Engine) 
      - registry defined on `RegistryProxyMixin`
      - instantiated via `this.buildRegistry();`
      - registry is an instance of `Registry`
        - defined at `ember.js/packages/@ember/-internals/container/lib/registry.ts` 
          - in here is where the bulk of the implementation of this RFC would live? (along with updating the type signatures of the methods all the way up the call tree (where not inferred))
