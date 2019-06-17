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

In order for `lookup` to be able to take a class definition as an argument, there will need to be an alternative way to _lookup_ instances of services by the class.

On the `Registry`, there already exists a reference to the class definition when registering an entry to the container.

```ts
registry.register('model:user', Person, {singleton: false });
registry.register('fruit:favorite', Orange);
registry.register('communication:main', Email, {singleton: false});
```

A new map can exist on the registry that can appropriately be wired up to register, unregister, etc to handle the lookup-by-class-definition at the same time as the current registry behaves, allowing for backward compatibility with todays injection usage.

Examples: 

(also pardon the naming, as it's for demonstration of the _roles_ that each class definition can have)
```ts
// This behaves as the base implementation
class MyInterface extends Service {
  @tracked foo = 0;

  add() {
    this.foo++;
  }
}

// registration would look like it is today
appInstance.register(MyInterface, MyInterface);
// where we register MyInterface on the *key* MyInterface,
//  just as today, it would look like
appInstance.register('service:my-interface', MyInterface);
```
Now, where this _IS_ Dependency Injection, and how we aren't just using the concrete class all the time is where you can do things like this

```ts
class MyImplementation extends MyInterface {
  add() {
    this.foo += 2;
  }
}

// both stubbing (in a test), or clobbering, would look the same
appInstance.register(MyInterface, MyImplementation);

const service = appInstance.lookup(MyInstance);

service instanceof MyImplementation // true
service instanceof MyInterface // true
```



## How we teach this

Instead of using strings or inferred injections, the guides should be updated to use the Class definition of a service that it intends to inject.


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
