---
Stage: Accepted
Start Date: 2020-10-04
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/680
---

# Implicit Injection Deprecation

## Summary

This RFC seeks to deprecate implicit injections on arbitrary Ember Framework objects. This is commonly done via `owner.inject` in an intializer or instance-initializer.

A prevalent example of implicit injection is found in `ember-data` [injecting](https://github.com/emberjs/data/blob/4bd2b327c4cbca831f9e9f8bc6b497200a212f9b/packages/-ember-data/addon/setup-container.js) their default `@ember-data/store` into all `@ember/routing/route` and `@ember/controller` factory objects. Here is a pruned example of how this looks.

```js
// app/initializers/store-inject.js
export function initialize(application) {
    application.inject('route', 'store', 'service:store');
    application.inject('controller', 'store', 'service:store');
}

export default {
    name: 'store-inject',
    initialize,
};
```

```js
export default class PostRoute extends Route {
  // This proposal seeks to make this service injection explicit
  // Currently users do not need to specify this injection if ember-data is installed
  // @service store;

  model() {
    return this.store.findRecord('post', 1);
  }
}
```

Ensuring a property is present on Ember's Framework objects should be explicitly defined. This will allow the Ember ecosystem to further progress the framework while easing the learning curve for all developers.

## Motivation

Implicit injections have long confused developers onboarding into Ember projects. Without an explicit injection visible on the class body, developers start to wonder how certain properties got there. We can infer from a common implicit injection in `ember-data` to reason about some of the downsides this presents. If a project has `ember-data` installed, an initializer at runtime registers the `@ember-data/store` on both
`@ember/routing/route` and `@ember/controller` factory objects.  This is a nice convenience for a project with many routes that are used for data fetching. However, it leads to many inconveniences that hinder learning and advancement of the framework. This includes:

1. By entangling specific objects with the whole dependency tree, we might be preventing incremental adoption. For example, `ember-data` has laid out its plan for adopting only specific packages in [Project Trim](https://github.com/emberjs/data/issues/6166). However, because `@ember-data/store` is injected by default on all `@ember/routing/route` and `@ember/controller` factory objects, a user cannot easily opt out of `@ember-data/store`. This is a common problem space in other communities as well. With previous JetBrains IDE installations, adding plugins required a reload. However, by removing a constructor function that injected all the dependencies at once, they were able to avoid reload after installation of a plugin.

2. Eager initialization of the dependency tree from which the property came from prevents tree shaking and incurs a performance hit. For example, if you app does not need `ember-data` to render the entry route, the `@ember-data/store` injection will take up more CPU cycles than necessary, hurting common user facing metrics.

3. Eager initialization makes it hard to users to write tests that need to take advantage of a stub. This is often a silent and annoying hurdle that users have to get around when writing tests. Moreover, this is especially apparent for users moving to Ember's new testing APIs in [RFC 232](https://github.com/emberjs/rfcs/blob/master/text/0232-simplify-qunit-testing-api.md) and [RFC 268](https://github.com/emberjs/rfcs/blob/master/text/0268-acceptance-testing-refactor.md), ensuring instance initializers run before each test.

4. Injection on commonly used framework objects can lead to deviation of project specific styleguides. For instance, injecting the `router` on the `@ember/component` factory object will certainly lead to instances where the `router` is explicitly injected and other instances where is it implicitly injected.

5. Users have come to associate `Route.model()` with a hook that returns `ember-data` models in the absense of explicit injection. While this can be true, it is not wholly true. New patterns of data loading are becoming accepted in the community including opting to fetch data in a Component.


## Detailed design

Removing implicit injections is not possible until we first issue deprecations. For example, `@ember/data` cannot remove its `owner.inject` without it being a breaking change. In doing so, deprecating first helps us maintain our strict semver commitment to not completing a major version bump and removing APIs without a deprecation path. As a result, it will take us 3 phases to remove `owner.inject`.

### 1. Deprecate implicit injection on target object

Simply deprecating `owner.inject` is not sufficient because it is very difficult to detect if you are relying on implicit injection. As a result, the first step will be issuing a deprecation on the target factory object that is receiving a property. This will surface the deprecation on user's owned objects including transitively through addons like `ember-data`. This can be accomplished by installing a native getter and setter on the target.

The _first time_ the property is accessed, either through getting or setting it, we will check if the property is explicitly injected. If it isn't, we will issue a deprecation in DEBUG builds with an until, id and url value to read more about this deprecation. In production, we will not issue this deprecation and will continue assigning the property. This deprecation will go `until: 4.0.0`.

To avoid this deprecation, you will need to explicitly add the injected property to the target object.  As an example, you can add `@service store` to your route or controller to avert this deprecation resulting from `@ember/data`'s use of `owner.inject`.  Users and addons can remove implicit injections as well.

Ember.js `v4.0.0` will still be released with `owner.inject`. However, it will not doing anything. Effectively a no-op.

### 2. Deprecate `owner.inject`.

The first phase did not actually deprecate the use of `owner.inject`.  As a result, we need to deprecate it's use directly iin `v4.0.0` before removing completely in `v5.0.0`.

For users and or addons, you likely already have removed implicit injections like `owner.inject` before upgrading to Ember.js `v4.0.0`.  However, for libraries like `@ember/data`, we will look to remove sometime in `v4.0.0` to resolve this new deprecation.

### 3. Profit!

Ember.js `v5.0.0` will finally remove the ability to call `owner.inject` to inject arbitrary properties on Ember's Framework objects.

It is important to consider the timeline of these three phases.  The first step will consist of a minimum of one release cycle.  Many addons and apps will need to make minor and major changes to their codebases before `v4.0.0`.

### Impementation Constraints

An important implementation detail of stage 1 of this RFC is that while we want
to issue a deprecation when the user relies on the implicit injection, we also
_must_ ensure that the implicit injection still "wins" and is assigned to the
value on the class, clobbering the explicit injection, for the time being. This
is the current behavior, and changing it, even in development builds, would be a
breaking change.

The proposed way to implement this is when an object with implicit injections is
created, for each implicit injection:

1. If the property already contains the value, do nothing as it has been setup
   eagerly by the user
2. If the property is a service injection, check that it is the correct service
   injection and if it is, do nothing and assign the value like normal
3. If the property is a native getter/setter, wrap the native getter/setter and
   the first time the value is accessed, check to see if it is the injected
   value. If it is, do nothing, otherwise log the deprecation.
4. If the property is not defined on the object or its prototype, install a
   getter/setter pair which log the deprecation the first time they are
   activated.

This should preserve the existing semantics while logging when the user is
accessing a value that was injected implicitly.

## How we teach this

The API docs would need to be overhauld in a few spots. First, we need to remove the [docs](https://guides.emberjs.com/release/applications/dependency-injection/#toc_factory-injections) about Factory injections. Second, we need to detail explicit injection where currently relying on implicit injection.  Many examples show fetching data via `this.store` but do not specifiy how `this.store` arrived as a property on the Route. Also apparent disclaimers for people visiting the `ember-data` docs that explicit injection of `@ember-data/store` is necessary and has deviated from past behaviour is likely prudent.

The default blueprints when generating a new Ember application will not change.

### Deprecation Guides

#### Stage 1

Implicit injections are injections that are made by telling Ember to inject a
service (or another type of value) into every instance of a specific type of
object. A common example of this was the `store` property that was injected into
routes and controllers when users installed Ember Data by default.

```js
export default class ApplicationRoute extends Route {
  model() {
    return this.store.findQuery('user', 123);
  }
}
```

Notice how the user can access `this.store` without having declared the store
service using the `@service` decorator. This was accomplished by using the
`owner.inject` API, usually in an initializer:

```js
export default {
  initialize(app) {
    app.inject('route', 'store', 'service:store');
    app.inject('controller', 'store', 'service:store');
  }
}
```

Implicit injections are difficult to understand, both because it's not obvious
that they exist, or where they come from.

In general, in order to migrate away from this pattern, you should use an
explicit injection instead of an implicit one. You can do this by using the
`@service` decorator wherever you are using the implicit injection currently.

Before:

```js
import { Route } from '@ember/routing/route';

export default class ApplicationRoute extends Route {
  model() {
    return this.store.findQuery('user', 123);
  }
}
```

After:

```js
import { Route } from '@ember/routing/route';
import { inject as service } from '@ember/service';

export default class ApplicationRoute extends Route {
  @service store;

  model() {
    return this.store.findQuery('user', 123);
  }
}
```

In some cases, you may be using an injected value which is not a service.
Injections of non-service values do not have a direct explicit-injection
equivalent. As such, to migrate away from these, you will have to rewrite the
injection to use services instead.

Before:

```js
// app/initializers/logger.js
import EmberObject from '@ember/object';

export function initialize(application) {
  let Logger = EmberObject.extend({
    log(m) {
      console.log(m);
    }
  });

  application.register('logger:main', Logger);
  application.inject('route', 'logger', 'logger:main');
}

export default {
  name: 'logger',
  initialize: initialize
};
```
```js
// app/routes/application.js
export default class ApplicationRoute extends Route {
  model() {
    this.logger.log('fetching application model...');
    //...
  }
}
```

After:

```js
// app/services/logger.js
import Service from '@ember/service';

export class Logger extends Service {
  log(m) {
    console.log(m);
  }
}
```
```js
// app/routes/application.js
import { inject as service } from '@ember/service';

export default class ApplicationRoute extends Route {
  @service logger;

  model() {
    this.logger.log('fetching application model...');
    //...
  }
}
```

In cases where it is not possible to convert a custom injection type into a
service, the value can be accessed by looking it up directly on the container
instead using the [lookup](https://api.emberjs.com/ember/3.22/classes/ApplicationInstance/methods/lookup?anchor=lookup)
method:

```js
// app/routes/application.js
import { getOwner } from '@ember/application';
import { inject as service } from '@ember/service';

export default class ApplicationRoute extends Route {
  get logger() {
    if (this._logger === undefined) {
      this._logger = getOwner(this).lookup('logger:main');
    }

    return this._logger;
  }

  set logger(value) {
    this._logger = value;
  }

  model() {
    this.logger.log('fetching application model...');
    //...
  }
}
```

You should always include a setter until the implicit injection is removed,
since the container will still attempt to pass it into the class on creation,
and it will cause errors if it attempts to overwrite a value without a setter.

### Stage 2

The `owner.inject` API would previously inject a values into every instance of a
particular type of class. For instance, you could inject the `store` service
automatically into every controller and route:


```js
export default {
  initialize(app) {
    app.inject('route', 'store', 'service:store');
    app.inject('controller', 'store', 'service:store');
  }
}
```

And in doing so, users could use the value without explicitly declaring it in
that type of object, using `@service`:

```js
import { Route } from '@ember/routing/route';

export default class ApplicationRoute extends Route {
  model() {
    return this.store.findQuery('user', 123);
  }
}
```

This ability was deprecated and removed in Ember v4.0.0, so users no longer can
do this, even if you are using the `owner.inject` API. It instead does nothing,
so you can safely remove it without any other changes necessary.

## Alternatives

- Continue with use of `owner.inject` but overhaul docs and recommend explicit injection.
- Provide application level super class with a default `@ember-data/store` injection provided by `ember-data` and other common examples in the community.
- Allow injection on a specific factory with a single backing Class instead of a generic factory object (e.g. `route` vs `route.index`).

## Open Questions

- Should we simply replace the [docs](https://guides.emberjs.com/release/applications/dependency-injection/#toc_factory-injections) on Factory Injections without an alternative?

## Related links and RFCs
- [Deprecate defaultStore located at `Route.store`](https://github.com/emberjs/rfcs/issues/377)
- [Pre-RFC: Deprecate implicit injections (owner.inject)](https://github.com/emberjs/rfcs/issues/508)
- [Deprecate implicit record loading in routes](https://github.com/emberjs/rfcs/issues/557)
