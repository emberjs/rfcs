- Start Date: 2019-03-14
- Relevant Team(s): Ember.js, Ember Data, Learning
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Injection Hook Normalization

## Summary

Standardize the basic injection hooks for Ember Octane's core framework objects:

1. Glimmer Component
2. Service
3. Route
4. Controller
5. Helper

This RFC would supercede the [Classic Class Owner Tunnel
RFC](https://github.com/emberjs/rfcs/pull/451) if it were to be accepted.

## Motivation

When the API design for Glimmer component finished up late last year, it was
decided that Glimmer component's `constructor` should receive the owner and
arguments directly as parameters, so injections and arguments would be available
to components during construction:

```js
export default class MyComponent extends Component {
  @service store;

  constructor(owner, args) {
    super(owner, args);

    this.store.queryRecord('person', 123);
  }
}
```

However, this has created a divergence with all other `EmberObject` based
classes, which do _not_ receive these values in the `constructor`, and instead
receive them just before `init` is triggered:

```js
export default class Store extends Service {
  @service fetch;

  constructor() {
    super();

    this.fetch; // this is undefined here
  }

  init() {
    super.init();

    this.fetch.get('api/config'); // it's defined here instead
  }
}
```

This divergence has brought up the more general question of how DI should assign
values during construction. Should we provide the `owner` to the constructor,
and require that users' clases set the owner themselves? Should we set the owner
after creation, and provide some kind of post-create hook? Should we try to
side-channel the owner somehow?

After researching other DI frameworks such as Spring, Guice, Dagger, Weaver,
Typhoon, and Angular.js, we've found there are two common ways to inject values:

1. Via explicit decoration of the constructor or the class.

   Spring:

   ```java
   public class MovieRecommender {

     private final CustomerPreferenceDao customerPreferenceDao;

     @Autowired
     public MovieRecommender(CustomerPreferenceDao customerPreferenceDao) {
       this.customerPreferenceDao = customerPreferenceDao;
     }

     // ...
   }
   ```

   Dagger:

   ```java
   class Thermosiphon implements Pump {
     private final Heater heater;

     @Inject
     Thermosiphon(Heater heater) {
       this.heater = heater;
     }

     ...
   }
   ```

   Angular.js:

   ```js
   import { Component } from '@angular/core';
   import { Hero } from './hero';
   import { HeroService } from './hero.service';

   @Component({
     selector: 'app-hero-list',
     template: `
       <div *ngFor="let hero of heroes">
         {{hero.id}} - {{hero.name}}
       </div>
     `,
   })
   export class HeroListComponent {
     heroes: Hero[];

     constructor(heroService: HeroService) {
       this.heroes = heroService.getHeroes();
     }
   }
   ```

2. Via decoration of properties, similar to Ember's current system:

   Dagger:

   ```java
   class CoffeeMaker {
     @Inject Heater heater;
     @Inject Pump pump;

     ...
   }
   ```

   Spring:

   ```java
   public class MovieRecommender {
       @Autowired
       private MovieCatalog movieCatalog;

       // ...
   }
   ```

Importantly, _no_ injections occur without explicit configuration or decoration
of some kind in any of the most mature DI frameworks, and injected properties
are _only_ available post construction. Most frameworks also provide
[post-creation and pre-destruction][1] hooks for setting up the instance and
tearing it down after all injections have been assigned.

[1]: https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#beans-postconstruct-and-predestroy-annotations

This RFC proposes that we normalize the assignment of the owner, and access to
injected properties, to follow the same conventions as other popular DI
frameworks. Injections will be assigned _after_ objects have been constructed,
and a `AFTER_INJECTION` symbol hook will be triggered after that. This will be
_approximately_ the same flow as `init` is currently for native classes, but is
a new hook to help with teaching purposes, since `init` has historically been
taught as the same as `constructor` in native classes (even though it is not).
We will also trigger a `BEFORE_DESTRUCTION` symbol hook when the container is
cleaning up objects. Finally, we'll create decorators that users can use to
decorate functions to run during these lifecycle hooks, so users have a friendly
and flexible way to define their class's lifecycles.

Doing this will ensure that users can rely on a consistent API for all DI
related classes, and that the lifecycle of the DI system will be teachable and
consisent. It also unblocks the usage of plain native classes which do not
extend a base class in the future. Finally , it will allow us to continue to
build on the DI system, and

## Detailed design

There are three parts to this design:

### Symbols and Decorators

Two symbols will be added to Ember:

- `AFTER_INJECTION`
- `BEFORE_DESTRUCTION`

These symbols will be publicly accessible, and will be the keys for the methods
that should be called on objects at the respective points in their lifecycles.
We will also add two decorators:

- `@afterInjection`
- `@beforeDestruction`

These decorators will be usable on methods, and will cause the methods to run
after injection and before destruction. Multiple methods can be decorated, and
will all run in the order that decorators are evaluated (from top to bottom in
the stage 1 spec).

### Glimmer Components

We'll update the `GlimmerComponent` class to _not_ receive any constructor
parameters. Instead, both `owner` and `args` will be assigned _after_ the object
has been fully constructed. We will also add the `AFTER_INJECTION` hook, which
will trigger immediately after `owner` and `args` have been assigned. The
`willDestroy` hook will also be renamed to `BEFORE_DESTRUCTION`. No default
implementation will be provided on the base class.

### `EmberObject` based classes (Route/Controller/Service/Helper)

In the case of `EmberObject` classes, `AFTER_INJECTION` will trigger _after_
`init`. Since the `owner` and arguments are assigned before `init`, it won't be
necessary to add any assignment code for these.

Importantly, _only_ classes that are constructed by the container will have
`AFTER_INJECTION` triggered - utility classes that are made by extending
`EmberObject` and used throughout user code will only have `init`. This is
because the new hook is specifically for interacting with injections after the
class has been fully constructed, and does not make sense in classes that are
not initialized by the container.

`BEFORE_DESTRUCTION` will trigger `destroy` in EmberObjects by default, since
the function currently runs at the same time. Users will be able to override
this function and call the super function using `super[BEFORE_DESTRUCTION]` or
`this._super()` in classic classes.

## How we teach this

We would teach these concepts along with our material on DI in general. We
primarily want to teach the `@afterInjection` and `@beforeDestruction`
decorators for native classes, and then teach the existence of the symbols for
classic class users.

An important point to cover will be the differences between `constructor` and
`@afterInjection` methods. We should teach `constructor` specifically as a
mechanism for _shaping_ the class, defining the basic fields and properties that
will exist on it. By contrast, methods decorated with `@afterInjection` are for
setting up state within the larger system - the Ember application. Concrete
examples can be used to drive this point home.

## Drawbacks

- The previously proposed Glimmer component API has been built and highly
  publicized. This change will definitely be surprising to some people, and may
  cause confusion. However, v1.0.0 of Glimmer component has _not_ been published
  yet, and we would not be breaking any semver guarantees. Additionally, there
  are likely not many early adopters, since we have not yet marked it as stable.

- There has been a lot of concern about confusion between `constructor` and
  `init` as we head further down the native class adoption path. This decision
  would formalize adopting `@afterInjection`, which is very similar to `init`,
  for _all_ classes for the forseeable future. This could also cause confusion,
  since container based classes would essentially always have this dichotomy of
  two "creation" hooks.

  Creating a different hook with a new name should help with this issue. `init`
  has historically been taught as the same as `constructor`, which is why some
  users assume it will have the exact same semantics, and are confused when it
  does not. Documenting the purpose for `@afterInjection`, along with clear,
  practical examples for when to use it, and when to use `constructor`, should
  go a long way toward preventing any additional confusion.

## Alternatives

- Standardizing on always passing the `owner` as the first parameter is another
  option here. This would bring us inline with the current behavior of Glimmer
  components, and a short term solution that would enable this is proposed in
  the [Classic Class Owner Tunnel RFC](https://github.com/emberjs/rfcs/pull/451).
