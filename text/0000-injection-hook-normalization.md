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
and a `didCreate` hook will be triggered after that. This will be
_approximately_ the same flow as `init` is currently for native classes, but is
a new hook to help with teaching purposes, since `init` has historically been
taught as the same as `constructor` in native classes (even though it is not).

Doing this will ensure that users can rely on a consistent API for all DI
related classes, and that the lifecycle of the DI system will be teachable and
consisent. It also unblocks the usage of plain native classes which do not
extend a base class in the future.

## Detailed design

There are two parts to this design:

1. **Glimmer Components**

   We'll update the `GlimmerComponent` class to _not_ receive any constructor
   parameters. Instead, both `owner` and `args` will be assigned _after_ the
   object has been fully constructed. We will also add the `didCreate` hook,
   which will trigger immediately after `owner` and `args` have been assigned.

2. **`EmberObject` based classes (Route/Controller/Service/Helper)**

   In the case of `EmberObject` classes, `didCreate` will trigger _after_
   `init`. Since the `owner` and arguments are assigned before `init`, it won't
   be necessary to add any assignment code for these.

   Importantly, _only_ classes that are constructed by the container will have
   `didCreate` triggered - utility classes that are made by extending
   `EmberObject` and used throughout user code will only have `init`. This is
   because the new hook is specifically for interacting with injections after
   the class has been fully constructed, and does not make sense in classes that
   are not initialized by the container.

In the future these hooks could be generalized into an underlying set of
primitives that would allow users to declaratively specify injections and hooks
with decorators:

```js
import { start, teardown } from "@ember/container";
import { tracked } from "@glimmer/tracked"
import { glimmer } from "@glimmer/component"

export default class ClockService {
  time = Date.now();
  #interval = undefined;

  @start
  start() {
    #interval = setInterval(() => this.time = Date.now(), 1000)
  }

  @tracked time;

  @teardown
  destroy() {
    clearInterval(#interval);
  }
}

export default @glimmer class MyComponent {
  constructor(@service clock) {
    this.createdAt = clock.time;
  }
}
```

However, this is beyond the scope of this RFC. For now, the design is focussed
on staying in-line with this approach, and avoiding polluting the `constructor`
signatures for the various base classes provided in Ember.

## How we teach this

We would teach these concepts along with our material on DI in general.
`didCreate` and its counter-part, `willDestroy`, should be thought of as
container hooks that can be used to setup and teardown state based on
injections.

An important point to cover will be the differences between `constructor` and
`didCreate`. We should teach `constructor` specifically as a mechanism for
_shaping_ the class, defining the basic fields and properties that will exist on
it. By contrast, `didCreate` is for setting up state within the larger system -
the Ember application. Concrete examples can be used to drive this point home.

## Drawbacks

- The previously proposed Glimmer component API has been built and highly
  publicized. This change will definitely be surprising to some people, and may
  cause confusion. However, v1.0.0 of Glimmer component has _not_ been published
  yet, and we would not be breaking any semver guarantees. Additionally, there
  are likely not many early adopters, since we have not yet marked it as stable.

- There has been a lot of concern about confusion between `constructor` and
  `init` as we head further down the native class adoption path. This decision
  would formalize adopting `didCreate`, which is very similar to `init`, for
  _all_ classes for the forseeable future. This could also cause confusion,
  since container based classes would essentially always have this dichotomy of
  two "creation" hooks.

  Creating a different hook with a new name should help with this issue. `init`
  has historically been taught as the same as `constructor`, which is why some
  users assume it will have the exact same semantics, and are confused when it
  does not. Documenting the purpose for `didCreate`, along with clear, practical
  examples for when to use it, and when to use `constructor`, should go a long
  way toward preventing any additional confusion.

## Alternatives

- Standardizing on always passing the `owner` as the first parameter is another
  option here. This would bring us inline with the current behavior of Glimmer
  components, and a short term solution that would enable this is proposed in
  the [Classic Class Owner Tunnel RFC](https://github.com/emberjs/rfcs/pull/451).

- This RFC approaches the current problem directly - how do we address the
  differences in behaviors of injections between different base classes in
  Ember today, and what is the way we want users to interact with injections in
  the long run?

  However, it is also touching on some larger underlying infrastructural debt.
  The reason this has become an issue at all is, in part, because Ember's
  container is heavily tied to the `EmberObject` model and its implementation
  details. There are several ways we could move forward with disconnecting
  these:

  1. We could move the container in a more annotation based direction, as
     described earlier in this RFC, and similarly to how other DI frameworks
     work.
  2. We could implement a more generic Manager layer, similar to Component and
     Modifier managers, that defines how a base class is managed by the
     container.
  3. We could do some combination of these systems.

  This would be a much larger refactor, with much more design required,
  especially if it would mean making more details of the container public. While
  this would be good for updating and rationalizing Ember's internals in the
  long run, these are implementation details that can still be addressed after
  this decision has been made. It is not necessary to make these decisions now
  to solve the basic question - should we pass injection parameters to the
  `constructor` (without explicit annotation), or should we rely on lifecycle
  hooks instead.
