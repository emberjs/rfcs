- Start Date: 2019-06-14
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: https://github.com/emberjs/rfcs/pull/502
- Tracking: (leave this empty)

# Explicit Dependency Injection

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

A new map can exist on the registry that can appropriately registers, unregisters, etc by class definition at the same time as the current registry behaves, allowing for backward compatibility with todays injection usage.


## How we teach this

Instead of using strings or inferred injections, the guides should be updated to use the Class definition of a service that it intends to inject.

Eventually, we'll want to introduce a deprecation for string injections to minimize different injection techniques.

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