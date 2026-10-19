---
stage: accepted
start-date: 2025-06-13T00:00:00.000Z
release-date:
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1111
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

# Deprecating `Ember.Evented` and `@ember/object/events`

## Summary

Deprecate the `Ember.Evented` mixin, the underlying `@ember/object/events` module (`addListener`, `removeListener`, `sendEvent`), and the `on()` function from `@ember/object/evented`.

## Motivation

For a while now, Ember has not recommended the use of Mixins. In order to fully
deprecate Mixins, we need to deprecate all existing Mixins of which `Evented` is one.

Further, the low-level event system in `@ember/object/events` predates modern
JavaScript features (classes, modules, native event targets, async / await) and
encourages an ad-hoc, implicit communication style that is difficult to statically
analyze and can obscure data flow. Removing it simplifies Ember's object model and
reduces surface area. Applications have many well-supported alternatives for
cross-object communication (services with explicit APIs, tracked state, resources,
native DOM events, AbortController-based signaling, promise-based libraries, etc.).

## Transition Path

The following are deprecated:

* The `Ember.Evented` mixin
* The functions exported from `@ember/object/events` (`addListener`, `removeListener`, `sendEvent`)
* The `on()` function exported from `@ember/object/evented`
* Usage of the `Evented` methods (`on`, `one`, `off`, `trigger`, `has`) when mixed into framework classes (`Ember.Component`, `Ember.Route`, `Ember.Router`)

Exception: The methods will continue to be supported (not deprecated) on the `RouterService`, since key parts of its functionality are difficult to reproduce without them. This RFC does not propose deprecating those usages.

### Recommended Replacement Pattern

Rather than mixing in a generic event emitter, we recommend refactoring affected code so that:

1. A service (or other long‑lived owner-managed object) exposes explicit subscription methods (e.g. `onLoggedIn(cb)`), and
2. Internally uses a small event emitter implementation. We recommend the modern promise‑based [emittery](https://www.npmjs.com/package/emittery) library, though any equivalent (including a minimal custom implementation) is acceptable.

This yields clearer public APIs, encapsulates implementation details, and makes teardown explicit by returning an unsubscribe function that can be registered with `registerDestructor`.

### Example Migration

Before (using `Evented`):

```js
// app/services/session.js
import Service from '@ember/service';
import Evented from '@ember/object/evented';
import { tracked } from '@glimmer/tracking';

export default class SessionService extends Service.extend(Evented) {
  @tracked user = null;

  login(userData) {
    this.user = userData;
    this.trigger('loggedIn', userData);
  }

  logout() {
    const oldUser = this.user;
    this.user = null;
    this.trigger('loggedOut', oldUser);
  }
}
```

```js
// app/components/some-component.js
import Component from '@glimmer/component';
import { inject as service } from '@ember/service';
import { registerDestructor } from '@ember/destroyable';

export default class SomeComponent extends Component {
  @service session;

  constructor(owner, args) {
    super(owner, args);
    this.session.on('loggedIn', this, 'handleLogin');
    registerDestructor(this, () => {
      this.session.off('loggedIn', this, 'handleLogin');
    });
  }

  handleLogin(user) {
    // ... update component state
  }
}
```

After (using `emittery`):

```js
// app/services/session.js
import Service from '@ember/service';
import { tracked } from '@glimmer/tracking';
import Emittery from 'emittery';

export default class SessionService extends Service {
  @tracked user = null;
  #emitter = new Emittery();

  login(userData) {
    this.user = userData;
    this.#emitter.emit('loggedIn', userData);
  }

  logout() {
    const oldUser = this.user;
    this.user = null;
    this.#emitter.emit('loggedOut', oldUser);
  }

  onLoggedIn(callback) {
    return this.#emitter.on('loggedIn', callback);
  }

  onLoggedOut(callback) {
    return this.#emitter.on('loggedOut', callback);
  }
}
```

```js
// app/components/some-component.js
import Component from '@glimmer/component';
import { inject as service } from '@ember/service';
import { registerDestructor } from '@ember/destroyable';

export default class SomeComponent extends Component {
  @service session;

  constructor(owner, args) {
    super(owner, args);
    const unsubscribe = this.session.onLoggedIn((user) => this.handleLogin(user));
    registerDestructor(this, unsubscribe);
  }

  handleLogin(user) {
    // ... update component state
  }
}
```

### Notes on Timing

Libraries like `emittery` provide asynchronous (promise‑based) event emission by default. Code which previously depended on synchronous delivery ordering may need to be updated. If strict synchronous behavior is required, a synchronous emitter (custom or another library) can be substituted without changing the public API shape shown above.

## Exploration

To validate this deprecation, we explored removal of the `Evented` mixin from Ember.js core (see: https://github.com/emberjs/ember.js/pull/20917) and confirmed that its usage is largely isolated and can be shimmed or refactored at the application layer.

## How We Teach This

* Update the deprecations guide (see corresponding PR in the deprecation app) with the migration example above.
* Remove most references to `Evented` from the Guides, replacing ad-hoc event usage examples with explicit service APIs.
* Emphasize explicit state and method calls, tracked state, resources, and native DOM events for orchestration.

## Drawbacks

* Applications relying heavily on synchronous event ordering may require careful refactors; asynchronous emitters change timing.
* Some addons may still expose `Evented`-based APIs and will need releases.
* Introduces a (small) external dependency when adopting an emitter library—though apps can implement a minimal sync emitter inline if desired.

## Alternatives

* Convert `Evented` to a decorator-style mixin (retains implicit pattern, less desirable).
* Keep `@ember/object/events` but deprecate only the mixin (adds partial complexity, limited long‑term value).
* Replace with a built-in minimal emitter utility instead of recommending third‑party (adds maintenance burden for Ember core).

## Unresolved Questions

* Do we want to provide (or document) a canonical synchronous emitter alternative for cases where timing matters?
* Should we explicitly codemod support (e.g. generate service wrapper methods) or leave migration manual?
* Any additional framework internals still relying on these APIs that require staged removal?
