---
Stage: Accepted
Start Date: 2022-05-23
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): "Ember.js, TypeScript,  Learning"
RFC PR: https://github.com/emberjs/rfcs/pull/821
---

# Public API for Type-Only Imports


## Summary <!-- omit in toc -->

Introduce public import locations for type-only imports which have previously had no imports, and fully specify their public APIs for end users:

- `Owner`, with `RegisterOptions` and `Factory`
- `Transition`
- `RouteInfo` and `RouteInfoWithAttributes`


## Outline <!-- omit in toc -->

- [Motivation](#motivation)
- [Detailed design](#detailed-design)
  - [`Owner`](#owner)
    - [`RegisterOptions`](#registeroptions)
    - [`Factory`](#factory)
    - [`FactoryManager`](#factorymanager)
    - [`FullName`](#fullname)
  - [`Transition`](#transition)
    - [`getOwner` and `setOwner`](#getowner-and-setowner)
  - [`RouteInfo`](#routeinfo)
    - [`RouteInfoWithAttributes`](#routeinfowithattributes)
- [How we teach this](#how-we-teach-this)
  - [`Owner`](#owner-1)
  - [`Transition`, `RouteInfo`, and `RouteInfoWithAttributes`](#transition-routeinfo-and-routeinfowithattributes)
  - [Blog post](#blog-post)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Unresolved questions](#unresolved-questions)


## Motivation

Prior to supporting TypeScript, Ember has defined certain types as part of its API, but *without* public imports, since the types were not designed to be imported, subclassed, etc. by users. For the community-maintained type definitions, the Typed Ember team chose to match that policy so as to avoid committing the main project to public API. With the introduction of TypeScript as a first-class language (see esp. RFCs [0724: Official TypeScript Support][0724] and [0800: TypeScript Adoption Plan][0800]) those types now need public imports so that users can reference them; per the [Semantic Versioning for TypeScript Types spec][spec], they also need to define *how* they are public: can they be sub-classed, re-implemented, etc.?

[0724]: https://rfcs.emberjs.com/id/0724-road-to-typescript
[0800]: https://rfcs.emberjs.com/id/0800-ts-adoption-plan
[spec]: https://www.semver-ts.org

Additionally, the lack of a public import or contract for the `Owner` interface has been a long-standing problem for *all* users, but especially TypeScript users, and given the APIs provided for e.g. the Glimmer Component class where `owner` is the first argument, the pervasive use of `getOwner` in low-level library code, etc., it is important for TypeScript users to be able to use an `Owner` safely, and for JavaScript users to be able to get autocomplete etc. from the types we ship.


## Detailed design

**Note:** For the terms "user-constructible" and "user-subclassable" see the [Semantic Versioning for TypeScript Types spec][spec].

### `Owner`

`Owner` is a **non-user-constructible** interface, with an intentionally minimal subset of the existing `Owner` API, aimed at what we *want* to support for `Owner` in the future:

```ts
export default interface Owner {
  lookup(fullName: FullName): unknown;

  register(
    fullName: FullName,
    factory: Factory<unknown> | object,
    options?: RegisterOptions
  ): void;

  factoryFor(fullName: FullName): FactoryManager<unknown> | undefined;
}
```

`Owner` is the default import from a new module, `@ember/owner`:

```ts
import type Owner from '@ember/owner';

function useOwner(owner: Owner) {
  let someService = owner.lookup('service:some-service');
  // ...
}
```

JS users can refer to it in JSDoc comments using `import()` syntax:

```js
/**
 * @param {import('@ember/owner').default} owner
 */
function useOwner(owner) {
  let someService = owner.lookup('service:some-service');
  // ...
}
```


`Owner` is non-user-constructible because constructing it correctly also requires the ability to provide a factory manager.[^existing-owner-usage]

In support of `Owner`, there are also four other newly-public types: `RegisterOptions`, `Factory`, `FactoryManager`, and `FullName`.

[^existing-owner-usage]: Existing usage of the Owner interface this way (e.g. setting custom owners for tests) mostly falls under the "intimate API" rules, and will likely be deprecated after a future introduction of a `createOwner()` hook so that that there is a public API way to get the required type.


#### `RegisterOptions`

`RegisterOptions` is a user-constructible interface:

```ts
export interface RegisterOptions {
  instantiate?: boolean | undefined;
  singleton?: boolean | undefined;
}
```

Although users will not usually need to use it directly, instead simply passing it as a POJO to `Owner#register`, it is available as a named export from `@ember/owner`:

```ts
import { type RegisterOptions } from '@ember/owner';
```

JS users can refer to it in JSDoc comments using `import()` syntax:

```js
/**
 * @param {import('@ember/owner').RegisterOptions} registerOptions
 */
function useRegisterOptions(registerOptions) {
  // ...
}
```


#### `Factory`

`Factory` is an existing concept available to users via [the `Engine#lookup` API][ff]. The public API only includes a `create` method, and we maintain that in this RFC. The result is this user-constructible interface:

[ff]: https://api.emberjs.com/ember/4.3/classes/EngineInstance/methods/factoryFor?anchor=lookup

```ts
export interface Factory<T> {
  create(initialValues?: Partial<T>): T;
}
```

`Factory` is available as a named import from `@ember/owner`:

```ts
import { type Factory } from '@ember/owner';

function useFactory(factory: Factory<unknown>) {
  let instance = factory.create();
}
```

JS users can refer to it in JSDoc comments using `import()` syntax:

```js
/**
 * @param {import('@ember/owner').Factory<unknown>} factory
 */
function useFactory(factory) {
  let instance = factory.create();
}
```

```ts
import { type Factory } from '@ember/owner';

class Person {
  name: string;
  age: number;

  private constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }
}

class PersonFactory implements Factory<Person> {
  create({ name = "", age = 0 } = {}) {
    return new Person(name, age);
  }
}
```

(This is not the actual usual internal implementation in Ember, but shows that it can be implemented safely with these types.)


#### `FactoryManager`

`FactoryManager` is an existing concept available to users via [the `Engine#factoryFor` API][ff]. The public API to date has included only two fields, `class` and `create`, and we maintain that in this RFC. The result is this **non-user-constructible** interface:

[ff]: https://api.emberjs.com/ember/4.3/classes/EngineInstance/methods/factoryFor?anchor=lookup

```ts
export interface FactoryManager<T> {
  readonly class: Factory<T>;
  create(initialValues?: Partial<T>): T;
}
```

`FactoryManager` is now available as a named import from `@ember/owner`:

```ts
import { type FactoryManager } from '@ember/owner';
```

JS users can refer to it in JSDoc comments using `import()` syntax:

```js
/**
 * @param {import('@ember/owner').FactoryManager} factoryManager
 */
function useFactoryManager(factoryManager) {
  // ...
}
```


#### `FullName`

The `FullName` type is a user-constructible alias for Ember’s string namespacing:

```ts
export type FullName = `${string}:${string}`;
```

This form allows both the namespaced (`namespace@type:name`) and non-namespaced (`type:name`) variants of these keys. It does not fully validate that these match Ember’s own internal rules for these types, but provides a bare-minimum check on the type safety of strings passed into `Owner` APIs.

Although users will not usually need to use it directly, instead simply passing it as a string literal to `Owner#lookup`, `Owner#register`, or `Owner#factoryFor`, it is available as a named import from `@ember/owner`:

```ts
import { type FullName } from '@ember/owner';
```

JS users can refer to it in JSDoc comments using `import()` syntax:

```js
/**
 * @param {import('@ember/owner').FullName} fullName
 */
function useFullName(fullName) {
  // ...
}
```


### `Transition`

`Transition` is a non-user-constructible, non-user-subclassable class. It is identical to the *existing* public API, with two new features:

- the specification of the generic type `T` representing the resolution state of the `Promise` associated with the route (and used with `RouteInfoWithAttributes`; see below)
- a public import location

Since it is neither constructible nor implementable, it should be supplied as type-only export. For example:

```ts
class _Transition<T = unknown>
  implements Pick<Promise<T>, 'then' | 'catch' | 'finally'> {

  data: Record<string, unknown>;
  readonly from: RouteInfoWithAttributes<T> | null;
  readonly promise: Promise<T>;
  readonly to: RouteInfo | RouteInfoWithAttributes<T>;
  abort(): Transition<T>;
  followRedirects(): Promise<T>;
  method(method?: string): Transition<T>;
  retry(): Transition<T>;
  trigger(ignoreFailure: boolean, name: string): void;
  send(ignoreFailure: boolean, name: string): void;

  // These names come from the Promise API, which Transition implements.
  then<TResult1 = T, TResult2 = never>(
    onfulfilled?: (value: T) => TResult1 | PromiseLike<TResult1>,
    onrejected?: (reason: unknown) => TResult2 | PromiseLike<TResult2>,
    label?: string,
  ): Promise<TResult1 | TResult2>;
  catch<TResult = never>(
      onRejected?: (reason: unknown) => TResult | PromiseLike<TResult>,
      label?: string,
  ): Promise<TResult | T>;
  finally(onFinally?: () => void, label?: string): Promise<T>;
}

export default interface Transition<T> extends _Transition<T> {}
```

It is the default import from `@ember/routing/transition`:

```ts
import type Transition from '@ember/routing/transition';
```

JS users can refer to it in JSDoc comments using `import()` syntax:

```js
/**
 * @param {import('@ember/routing/transition').default} theTransition
 */
function takesTransition(theTransition) {
  // ...
}
```

#### `getOwner` and `setOwner`

Both of the existing `getOwner` and `setOwner` functions now make *much* more sense as named exports from `@ember/owner`. They will also now take `owner` as the type of their argument explicitly:

```ts
export function getOwner(object): Owner | undefined;
export function setOwner(object: unknown, owner: Owner): void;
```

The existing exports from `@ember/application` will become re-exports of these functions. The existing exports will also be deprecated, with the deprecation becoming "Available" no earlier than the first minor *after* an LTS containing the updated import location. For example, if `getOwner` and `setOwner` are available to import from `@ember/owner` in Ember v4.7, the deprecation for the imports from `@ember/application` would not be deprecated until at least Ember v4.9, after Ember v4.8 LTS.[^timeline]

[^timeline]: This is in line with our normal approach to deprecation: it allows the addon ecosystem to absorb the change via LTS support releases, so that consuming apps are not flooded with deprecations without recourse.


### `RouteInfo`

`RouteInfo` is a non-user-constructible interface. It is identical to the *existing* public API, with the addition of a public import.

```ts
export default interface RouteInfo {
  readonly child: RouteInfo | null;
  readonly localName: string;
  readonly name: string;
  readonly paramNames: string[];
  readonly params: Record<string, string | undefined>;
  readonly parent: RouteInfo | null;
  readonly queryParams: Record<string, string | undefined>;
  readonly metadata: unknown;
  find(
    callback: (item: RouteInfo, index: number, array: RouteInfo[]) => boolean,
    target?: unknown
  ): RouteInfo | undefined;
}
```

It is the default import from `@ember/routing/route-info`:

```ts
import type RouteInfo from '@ember/routing/route-info';
```

JS users can refer to it in JSDoc comments using `import()` syntax:

```js
/**
 * @param {import('@ember/routing/route-info').default} routeInfo
 */
function takesRouteInfo(routeInfo) {
  // ...
}
```


#### `RouteInfoWithAttributes`

`RouteInfoWithAttributes` is a non-user-constructible interface, which extends `RouteInfo` by adding the `attributes` property. The attributes property represents the resolved return value from the route's model hook:

```ts
export interface RouteInfoWithAttributes<T = unknown> extends RouteInfo {
  attributes: T;
}
```

It is a named export from `@ember/routing/route-info`:

```ts
import { type RouteInfoWithAttributes } from '@ember/routing/route-info';
```

JS users can refer to it in JSDoc comments using `import()` syntax:

```js
/**
 * @param {import('@ember/routing/route-info').RouteInfoWithAttributes} routeInfo
 */
function takesRouteInfoWithAttributes(routeInfoWithAttributes) {
  // ...
}
```


## How we teach this

These concepts all already exist, but need updates to and in some cases wholly new pages in Ember's API docs.


### `Owner`

- We need to introduce API documentation in a new, dedicated module for `Owner`, `@ember/owner`. The docs on `Owner` itself should become the home for much of the documentation currently shared across the `EngineInstance` and `ApplicationInstance` classes.

- We need to document the relationship between `EngineInstance` and `ApplicationInstance` as implementations of `Owner`.

- We must update the existing docs for `factoryFor` to clarify that it is the only public API for getting a `FactoryManager`.


### `Transition`, `RouteInfo`, and `RouteInfoWithAttributes`

These types need two updates to the existing API documentation:

- Specify the modules they can be imported from (as noted above).

- Specify that these types are *not* meant to be implemented by end users. For example:

  > While the `Transition` interface can be imported to name the type for use in your own code, it is not user-implementable. The only supported way to get a `Transition` is by using Ember's routing APIs, like [the `transitionTo` method](https://api.emberjs.com/ember/4.3/classes/RouterService/methods/transitionTo?anchor=transitionTo) on [the router service](https://api.emberjs.com/ember/4.3/classes/RouterService).

  And:

  > While the `RouteInfo` interface can be imported to name the type for use in your own code, it is not user-implementable. The only supported way to get a `RouteInfo` is using a public API which provides it, including but not limited to:
  >
  > - the `from` or `to` properties on an instance of [the `Transition` class](https://api.emberjs.com/ember/4.3/classes/Transition)
  > - the `currentRoute` property on [the `RouterService` class](https://api.emberjs.com/ember/4.4/classes/RouterService)
  > - the `child` or `parent` properties on another `RouteInfo` instance, or as returned from the `find()` method on a `RouteInfo`


### Blog post

Besides the API docs described above and the usual discussion of new features in an Ember release blog post, we will include an extended discussion in that blog post, a blog post timed to come out around the same time as that release, or a blog post corresponding to when we add them to DefinitelyTyped, as makes the most sense. That post will situate these as part of the work done on the "road to TypeScript":

- emphasizing the benefits to both JS and TS users

- providing a straightforward explanation of the "non-user-constructible" constraint and refer users to the Semantic Versioning for TypeScript Types spec for more details

- explaining some choices about the public `Owner` API:
    - that it includes the subset of the `Owner` API we want to maintain over time, *not* the full set available on the `EngineInstance` type, much of which we expect to deprecate over the course of the 4.x release cycle
    - that it does not include type safe registry look-ups, since we are waiting to see what TypeScript does with the Stage 3 decorators spec before deciding how to work with registries going forward[^revise]

[^revise]: Depending on implementation timelines, it is possible we will revise this based on what TypeScript ships in the meantime.

More general work to clarify the use of TypeScript with these APIs will be addressed as part of the forthcoming RFCs on integrating TypeScript into Ember's docs.


## Drawbacks

For once: none! These are already all "intimate API" *at minimum*—`Owner` is effectively public API—and the only difference between the _status quo_ and the proposed outcome is that users can know these imports won't break. Moreover, the design here is forward-compatible with any iterations on these types we can currently foresee, including expanding the type of `Owner#lookup` to support registry, changes to the `Owner` API per [RFC #0585: Improved Ember Registry APIs][0585], or future directions for the Ember routing layer.

[0585]: https://rfcs.emberjs.com/id/0585-improved-ember-registry-apis


## Alternatives

- We could leave these in their private API locations.
- We could not publish these types at all, and have users continue to use utility types to name them (`ReturnType` etc.).


## Unresolved questions

- Should `Owner` be exported from `@glimmer/owner` instead? Presently, that acts as only a pass-through, with the notion of an owner being delegated to implementors of the various Glimmer "manager" APIs..

- Should `Owner` have any other of its existing APIs, and in particular should it be identical to the APIs exposed via `EngineInstance`, including these additional methods?
    - `inject`
    - `ownerInjection`
    - `registerOptions`
    - `registerOptionsForType`
    - `registeredOption`
    - `registeredOptions`
    - `registeredOptionsForType`
    - `unregister`

    (Since `Owner` itself has never been public API, and we hope to *deprecate and remove* all of these methods in the 4.x → 5.0 era, it seems best to leave them off of `Owner` in the newly-public API, only publishing the interface we want to support long-term.


- Should `Owner` be directly user-constructible, or should users be restricted to subclassing one of the existing concrete instances (`Engine` or its subclass `Application`)?
