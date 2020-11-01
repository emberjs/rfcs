---
Start Date: 2020-01-27
Relevant Team(s): Ember.js, Learning
RFC PR: #0585
Tracking: (leave this empty)

---

# Improved Ember Registry APIs

## Summary

Introduce a new, object-based API for all registry APIs; deprecate the current string-based microsyntax registry APIs; and introduce a `capabilities` property to the resolver to safely support existing resolvers. Extend the new API for service and controller injections, and deprecate their microsyntax for namespaced injections.

Today the registry APIs are all of shapes roughly like this:

```js
getOwner(this).lookup('service:session');
```

This RFC proposes that they would instead be written like this:

```js
getOwner(this).lookup({ type: 'service', name: 'session' })
```

Service and controller injections which include a namespace currently look like this:

```js
class Example {
  @service('global@session')
  session;
}
```

This RFC proposes that these would instead be written like this:

```js
class Example {
  @service({ namespace: 'global', name: 'session' })
  session;
}
```

### Table of Contents

- [Summary](#summary)
  - [Table of Contents](#table-of-contents)
- [Motivation](#motivation)
  - [Microsyntax problems](#microsyntax-problems)
  - [TypeScript problems](#typescript-problems)
  - [Performance benefits?](#performance-benefits)
- [Detailed design](#detailed-design)
  - [`Identifier` types](#identifier-types)
  - [`Resolver`](#resolver)
  - [Owner APIs](#owner-apis)
    - [Options](#options)
    - [`Factory` and `FactoryManager`](#factory-and-factorymanager)
    - [`Owner` API diff](#owner-api-diff)
  - [Service and Controller Injections](#service-and-controller-injections)
  - [Codemod](#codemod)
  - [Deprecation messaging](#deprecation-messaging)
    - [Guide](#guide)
    - [In-app](#in-app)
  - [Rollout](#rollout)
- [How we teach this](#how-we-teach-this)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [Supply just `schema` instead of `capabilities`](#supply-just-schema-instead-of-capabilities)
  - [A new String-based API](#a-new-string-based-api)
  - [String-based API as “sugar”](#string-based-api-as-sugar)
  - [Object-based API](#object-based-api)
  - [Do nothing](#do-nothing)
- [Unresolved questions](#unresolved-questions)
- [Appendix: TypeScript](#appendix-typescript)
  - [TypeScript motivation](#typescript-motivation)
  - [Proposed type definitions](#proposed-type-definitions)

## Motivation

There are two primary motivations here: replacing the string-based microsyntax with an idiomatic JavaScript API, and making the API more amenable to correct types for TypeScript users.

### Microsyntax problems

The current design has worked well enough for a long time, but it adds conceptual overhead to learning how to use Ember. JavaScript has a lightweight and easy way of defining sets of related data: plain old JavaScript objects. By contrast, the existence of this Ember-specific microsyntax requires users to learn a new string-based syntax (which is *not* used elsewhere in the JavaScript community) and internalize how it works when they first encounter the registry APIs.

This need often comes relatively early in the learning process: the first time a user needs to write a unit test for a service. We currently devote [an entire section of the guides](https://guides.emberjs.com/release/applications/dependency-injection/#toc_factory-registrations) to explaining how factory registrations work, including a paragraph devoted to explaining the microsyntax. A normal JavaScript API would simplify this entire section.

Changing to a normal JavaScript-object-based API means tooling can provide in-editor benefits around autocompletion for the API, which is simply not possible for the microsyntax. Finally, a non-microsyntax API will be more amenable to future codemods if this should need to change in the future.

### TypeScript problems

For TypeScript users, the current API is not type-safe, and can be made so only with considerable extra work by developers and the Typed Ember maintainers. Blueprint maintenance and end users’ mental overhead associated with existing solutions for Ember’s stringly-typed APIs would effectively have to *double* to provide type safety for today’s registry APIs. For details, see [**Appendix: Typescript**](#appendix-typescript).

### Performance benefits?

Finally, there *may* be some very small performance wins here, since in this schema there is no need to parse a string. This is not considered to be a significant motivator, however, since the tradeoffs between object allocation (memory utilization) and string parsing (CPU utilization) are likely to be largely irrelevant in practice for this API.

## Detailed design

Every registry API will be updated to take a new `Identifier` type in place of the string microsyntax, while maintaining backwards compatibility by introducing versioning to the resolver. A codemod will be provided to allow users to migrate directly to the new API. The legacy resolver APIs are deprecated until Ember 4.0.

(TypeScript consumers should see [**Proposed Type Definitions**](#proposed-type-definitions) in the appendix for additional details on type safety and resolution.)

<i>**Note:** the diff views used throughout are meant to highlight the new APIs compared to the old, *not* to indicate the removal of the deprecated items. They will only be removed at Ember 4.0.</i>

### `Identifier` types

The core new type in the design of the API is an *identifier*, which is always passed as the first argument to resolver APIs. There are three kinds of identifiers: `FactoryIdentifier` and `FactoryTypeIdentifier` (discussed here), and `InjectionIdentifier` (discussed below in [**Session and Controller Injections**](#service-and-controller-injections)).

```ts
interface FactoryIdentifier {
  type: string;
  name: string;
  namespace?: string;
}
```

Here the `type` corresponds to the prefix component of the legacy API, and `name` to the postfix component of the legacy API: `service:foo` becomes `{ type: 'service', name: 'foo' }`.

The `namespace` field allows for identifiers to be restricted to a particular namespace, as in looking up a resolution from an addon. It is optional since it is not required in normal usage, as is the case in the microsyntax today, where it may be provided via a prefix and delimited with `@`: `<namespace>@<type>:<name>`. This also supports experimental tools which namespace lookups for addons (e.g. [ember-holy-futuristic-template-namespacing-batman](https://github.com/rwjblue/ember-holy-futuristic-template-namespacing-batman)).

We also introduce `FactoryTypeIdentifier` to distinguish between injections for *all factories of a given type* (`FactoryTypeIdentifier`) and *specific factories* (`Identifier`):

```ts
interface FactoryTypeIdentifier {
  type: string;
}
```

As is the case in the microsyntax-based design, these factory type injections may not be namespaced.

### `Resolver`

Ember’s `Resolver` function is public API, designed to be customized and overridden. To support backwards compatibility with existing custom resolvers, and following the lead of the [Custom Components RFC (#0213)](https://github.com/emberjs/rfcs/blob/master/text/0213-custom-components.md#capabilities) and the [Element Modifiers RFC (#0373)](https://github.com/emberjs/rfcs/blob/master/text/0373-Element-Modifier-Managers.md#capabilities), we introduce a `capabilities` property to the `Resolver` API, which may be `undefined` or the return value of a new `capabilities` function, which defines a `Capabilities` type.

The `capabilities` function is a public export of the `@ember/application` module:

```ts
interface OptionalCapabilities {
  modulesBased?: boolean;
  disableMicrosyntax?: boolean;
}

export function capabilities(compatVersion: '3.18', capabilities?: OptionalCapabilities): unknown;
```

The return type of `capabilities` is `unknown` for public consumers; the actual return type is private API and should not be relied on by implementors of custom resolvers.

The `modulesBased` capability rationalizes the existing `moduleBasedResolver` property on the `Resolver` class, integrating it into the new `capabilities` API.

During the 3.x Ember series, if `capabilities` is `undefined`, the legacy behavior of the resolver is maintained. The signature of the `Resolver` type is therefore:

```ts
interface Resolver {
  capabilities?: Capabilities;
  resolve(identifier: string | FactoryIdentifier);
}
```

After Ember 4.0, the type a custom resolver must implement will be updated so that `capabilities` will be required and `resolve` will only accept a `FactoryIdentifier`:

```ts
interface Resolver {
  capabilities: Capabilities;
  resolve(identifier: FactoryIdentifier);
}
```

This design is completely backwards compatible: it continues working exactly as it does today. (See [**Detailed Design > Rollout**](#rollout) below for details.) Until 4.0, both apps and addons will be able to use the microsyntax-based API *or* the new identifier-based API. It also allows for clean interop for existing custom resolvers.

Besides the recent use of capabilities in the modifier and component managers, there is also some prior art for this kind of versioning applied to the resolver specifically: [a similar “stamp”](https://github.com/emberjs/ember.js/pull/9994) was used for accomodating different behaviors when using the modules-based resolver vs. not (back in 2014)!

### Owner APIs

The owner APIs all change to use `Identifier` (or `FactoryTypeIdentifier` as appropriate) instead of strings. 

Throughout, for the purposes of registration lookup, we use deep object value equality, *not* object identity. This specifically applies to:

- `Owner.factoryFor`
- `Owner.lookup`
- `Owner.registerOptions`
- `Owner.registerOptionsForType`
- `Owner.registeredOption`
- `Owner.registeredOptions`
- `Owner.registeredOptionsForType`
- `Owner.resolveRegistration`

####  Options

Some `Owner` registry APIs take lookup or registration options. This RFC does not propose changing these options in any way, but their definitions are provided here for completeness:

```ts
export interface LookupOptions {
  singleton?: boolean;
  instantiate?: boolean;
}

interface RegisterOptions {
  singleton?: boolean;
  instantiate?: boolean;
}
```

#### `Factory` and `FactoryManager`

Some APIs refer to *factories* and *factory managers*. The API for these classes adds a new `identifier` field and deprecates the `fullName` and `normalizedName` fields:

```diff
 interface FactoryClass {
   positionalParams?: string | string[] | undefined | null;
 }
 
 interface Factory<T, C extends FactoryClass | object = FactoryClass> {
   class?: C;
-  fullName?: string; // DEPRECATED
-  normalizedName?: string; // DEPRECATED
+  identifier: FactoryIdentifier;
   create(props?: { [prop: string]: any }): T;
 }
 
 interface FactoryManager<T = object> {
   readonly class: Factory<T>;
-  readonly fullName?: string; // DEPRECATED
-  readonly normalizedName?: string; // DEPRECATED
+  identifier: FactoryIdentifier;
   create(props?: object): T;
 }
```

#### `Owner` API diff

```diff
 interface Owner {
-  factoryFor(fullName: string, options: LookupOptions): FactoryManager;
+  factoryFor(identifier: FactoryIdentifier, options: LookupOptions): FactoryManager;

-  hasRegistration(fullName: string): boolean;
+  hasRegistration(identifier: FactoryIdentifier): boolean;

-  inject(factoryNameOrType: string, property: string, injectionName: string): void;
+  inject(factory: FactoryIdentifier | FactoryTypeIdentifier, property: string, injection: FactoryIdentifier): void;

-  lookup(fullName: string, options?: LookupOptions): any;
+  lookup(identifier: FactoryIdentifier, options?: LookupOptions): any;

-  register(fullName: string, factory: any, options?: RegisterOptions): void;
+  register(identifier: FactoryIdentifier, factory: any, options?: RegisterOptions): any;

-  registerOptions(fullName: string, options: RegisterOptions): void;
+  registerOptions(identifier: FactoryIdentifier, options: RegisterOptions): any;

-  registerOptionsForType(fullName: string, options: RegisterOptions): void;
+  registerOptionsForType(identifier: FactoryTypeIdentifier, options: RegisterOptions): any;

-  registeredOption(fullName: string, optionName: string): RegisterOptions;
+  registeredOption(identifier: FactoryIdentifier, optionName: string): RegisterOptions;

-  registeredOptions(fullName: string): RegisterOptions;
+  registeredOptions(identifier: FactoryIdentifier): RegisterOptions;

-  registeredOptionsForType(type: string): RegisterOptions;
+  registeredOptionsForType(type: FactoryTypeIdentifier): RegisterOptions;

-  resolveRegistration(type: string): Factory;
+  resolveRegistration(identifier: FactoryIdentifier): Factory;

-  unregister(fullName: string): void;
+  unregister(identifier: FactoryIdentifier): void;
 }
```

### Service and Controller Injections

The other major examples of registry interactions are service and controller injections. This RFC proposes to normalize both to support the same identifier-based lookup as the `Owner` APIs.

We introduce an `InjectionIdentifier` type, representing the service or controller to be injected:

```ts
interface InjectionIdentifier {
  name: string;
  namespace?: string;
}
```

The `inject` (whether `as service` or `as controller`) function now accepts *either* a string representing the local injection name (sans namespace) *or* an `InjectionIdentifier`:

- **Service:**
    ```ts
    export function inject(name: string): Service;
    export function inject(identifier: InjectionIdentifier): Service;
    ```

- **Controller:**
    ```ts
    export function inject(name: string): Controller;
    export function inject(identifier: InjectionIdentifier): Controller;
    ```

The `'<namespace>@<name>'` form is deprecated and will be removed at 4.0. The no-argument form, and the name-only string form are still allowed. Users *may* invoke using `{ name: '<name>' }` but are not required to.

- **Good** (using services as an example):

    ```js
    import { inject as service } from '@ember/service';

    export default class Example {
      @service foo;
      
      @service('bar')
      barRenamed;
      
      @service({ name: 'quux' })
      quuxViaIdentifier;
      
      @service({ namespace: 'baz', name: 'neato' })
      neatoNamespaced;
    }
    ```

- **Deprecated:**

    ```js
    import { inject as service } from '@ember/service';

    export default class Example {
      @service('baz@neato') neatoNamespaced;
    }
    ```

### Codemod

All existing *static* microsyntax invocations can be straightforwardly migrated to the new syntax with a codemod. Some dynamic invocations will not be possible to migrate this way.

For example, this (in a test) is migrate-able:

```diff
- this.owner.lookup('service:session');
+ this.owner.lookup({ type: 'service', name: 'session' });
```

But this is not:

```js
function buildLookup(type, name) {
  return `${type}:${name}`;
}

this.owner.lookup(buildLookup('service', 'session'));
```

The majority of uses are likely to be codemoddable, but not *all* will.

### Deprecation messaging

#### Guide

> #### Deprecate registry string-based microsyntax
> ##### until: 4.0.0
> ##### id: ember-resolver.string-based-microsyntax
> 
> Ember has historically supported a string-based microsyntax of the format `'<namespace>@<type>:<name>'` (where both `<namespace>@` and `:<name>` are optional) for registering, resolving, looking up, and injecting items into Ember’s dependency injection container. These have been replaced with an object-based API where each part of the registry identifier is named explicitly.
> 
> You should replace calls using the string-based microsyntax with the new API. For example, these lookups in a test context:
> 
> ```js
> this.owner.lookup('service:session');
> this.owner.register(
>   'shared@service:clipboard',
>   class Clipboard {
>     copyToClipboard(source) {/* no-op */}
>   }
> )
> this.owner.inject('controller', 'session', 'service:session');
> ```
>
> Should be changed to:
> 
> ```js
> this.owner.lookup({ type: 'service', name: 'session' });
> this.owner.register(
>   { namespace: 'shared', type: 'service', name: 'clipboard' },
>   class Clipboard {
>     copyToClipboard(source) {/* no-op */}
>   }
> );
> this.owner.inject(
>   { type: 'controller' },
>   'session',
>   { type: 'service', name: 'session' }
> );
> ```

#### In-app

Invocation of v0 resolver APIs which require a `FactoryIdentifier` will trigger the following deprecation message (using `lookup` as an example):

> You invoked `Owner.lookup` with a `string` full name: `'<type>:<name>'`. This usage is deprecated and will be removed in Ember 4.0. Instead, pass an object identifier: `{ type: '<type>', name: '<name>' }`.

Here, the message should substitute the *actual* passed type and name. For example, if the user tried to look up a service named `session` by invoking `lookup('service:session')` the deprecation will read:

> You invoked `Owner.lookup` with a string full name: `'service:session'`. This usage is deprecated and will be removed in Ember 4.0. Instead, pass an identifier object: `{ type: 'service', name: 'session' }`.

If the user tried to look up a service named `session` from the `shared` namespace by invoking `lookup('shared@service:session')` the deprecation will read:

> You invoked `Owner.lookup` with a string full name: `'shared@service:session'`. This usage is deprecated and will be removed in Ember 4.0. Instead, pass an identifier object: `{ namespace: 'shared', type: 'service', name: 'session' }`.

For methods which require a `FactoryTypeIdentifier`, the wording is adjusted appropriately:

> You invoked `Owner.lookup` with a string type name: `'service'`. This usage is deprecated and will be removed in Ember 4.0. Instead, pass a factory type identifier object: `{ type: 'service' }`.

Methods which support either `Identifier`s or `FactoryTypeIdentifier`s should display the appropriate variant of the message according to what the user actually supplied.

### Rollout

The rollout will be phased. For both Ember’s `Resolver` API and the default resolver supplied with `ember-resolver`:

- The new API will be introduced in a minor release as normal, *without* introducing the deprecation warning for the microsyntax-based design, and *including* the two codemods required for this API change: one for all Ember users, and one for TypeScript users (see [**Proposed Type Definitions**](#proposed-type-definitions) in the [**TypeScript Appendix**](#appendix-typescript) below.)

- After *at least* one minor version, the deprecation message will be introduced. This will give addons time to adopt the new API *before* their consumers start experience deprecation noise from addon usage.

For Ember only:

- We will supply a polyfill for the new behavior, supporting *at least* the current LTS. The behavior here can be polyfilled to many if not all existing LTS releases; we would likely limit this slightly, but would support *many* LTS releases.

## How we teach this

- **New concepts:** this RFC introduces the concept of an identifier through the `FactoryIdentifier`, `FactoryTypeIdentifier`, and `InjectionIdentifier` types. These define the arguments to the various registry functions. The basic shapes of these objects, and perhaps their names (as inline text like “factory identifier,” “factory type identifier,” and “injection identifier”) will need to be integrated into the guides and API docs. See also [**Unresolved Questions**](#unresolved-questions): there may be other names which work better.

- The existing documentation about dependency injection can unchanged apart from simplifications in certain key areas. Paragraphs explaining the microsyntax can be eliminated or simplified by showing the identifier object types. New Ember users will continue to benefit from the detailed introduction to the concepts in the guides, but will have one fewer concept to learn along the way.

- All instances of the API documentation will need to be updated to reflect the changes to their signatures.

- The deprecation message must be added to Ember’s deprecation page and tooling.

- For the duration of Ember 3.x releases, the Guides and API docs will need explanations of the previous, deprecated APIs. These explanations should be brief, focused on how to migrate away from them preferably via the supplied codemod, rather than diving into the details of how the legacy system worked.

## Drawbacks

- The existing APIs have served Ember well for a long time. Introducing any new change involves some risk, particularly in areas so core to how Ember behaves.

- The proposed API here deprioritizes brevity in favor of clarity, readability, maintainability, tooling support, and being “normal” JavaScript. `'service:session'` is many fewer characters to type than `{ type: 'service', name: 'session' }`. (While many people highly value brevity and we should avoid overly verbose APIs, long experience suggests that *reading* and *changing* are more important than *first writing* code.)

## Alternatives

### Supply just `schema` instead of `capabilities`

Instead of the manager-inspired `capabilities` API, we could implement a simpler design with simply supplies a `schemaVersion` (or just `schema`) property on the `Resolver` type. This is less flexible, but also simpler.

```ts
interface ResolverV0 {
  schemaVersion?: 0;
  resolve(fullName: string);
}
```

If `schemaVersion` is `1`, the `resolve` function has a new signature, using the `FactoryIdentifier` type introduced above:

```ts
interface ResolverV1 {
  schemaVersion: 1;
  resolve(identifier: FactoryIdentifier)
}
```

The public API is *either* of the Resolver variants:

```ts
type Resolver = ResolverV0 | ResolverV1;
```

The other registry API changes may also be implemented in terms of the new resolver `schemaVersion` check.

### A new String-based API

Instead of providing an object-based API, we could provide a new string-based API. The `lookup` method, then, would have a signature like this:

```ts
interface Owner {
  lookup(type: string, name: string, options: Options);
}
```

This has the advantage of being a smaller diff from today’s world, and cutting down on the extra typing of the new API design:

```diff
- lookup('service:foo')
+ lookup('service', 'foo')
```

vs.

```diff
- lookup('service:foo');
+ lookup({ type: 'service', name: 'foo' })
```

This is a less flexible proposal, and—more importantly—it has a significant downside for usability beyond the simplest case. Supporting namespaces here would require *another* string argument, and functions with multiple arguments of the same type are notoriously easy to misuse. (“Wait, what’s the order? Namespace first or type first? Does namespace come after type, or after name?” etc.)

### String-based API as “sugar”

While this RFC recommends the object-based API as the *primary* API, we could supply the string-based API outlined above for the simple and most common case, so that this—

```js
lookup('service', 'foo')
```

—would be equivalent to this:

```js
lookup({ type: 'service', name: 'foo'})
```

This could be nice from an ergonomics perspective, and introduces only a small amount of extra complexity in the implementation: the resolver `capabilities` key still allows the implementation to distinguish between the `lookup('<type>:<name>')` form and the `lookup(type, name)` form.

However, it does require introspection on the arguments passed to the functions (extra implementation overhead), and it comes with confusion and arguments about which form to use: should users prefer the string based version where it is possible, or default to the object version for consistency? It would also means changes (e.g. to introduce a namespace) would become large and more error prone. Finally, as with the alternative where this is the *only* API, it is still less explicit and clear; users have to *remember* that `type` precedes `name`, whereas the object-form eliminates that issue.

### Object-based API

Another brief syntax might use the type as the key and the name as its value:

```js
lookup({ service: 'foo' })
```

While this initially seems nice from the perspective of the consumer of the API, it muddies the API substantially and makes the implementation worse as well. 

On the API front, for example, what would this mean?

```js
lookup({ service: 'foo', route: 'bar' });
```

The API proposed by the RFC avoids this confusing scenario entirely!

As far as implementation goes, this would be more complex and have much worse performance: the lookup would have to iterate over all keys on the object passed every time, mapping those keys to the *known* identifer interface and then checking the others against the registry. This also makes the TypeScript implementation considerably more difficult, for similar reasons to the runtime implementation, but at compile time. (Implementer concerns should not be *primary*, but they are important, and here they have performance implications that, while not dramatic, can easily be avoided by other proposals.)

### Do nothing

Leaving the API as it is remains an option, with both the upsides and downsides of the _status quo_.

## Unresolved questions

- Is `@ember/application` the appropriate home for the new exports? Or should they liver somewhere else, such as `@ember/resolver` or `@ember/application/resolver`?

- Is `FactoryIdentifier` the right name for the primary identifier interface? `Identifier` is shorter but overlaps with the notion of identifiers from Ember Data and is perhaps overly generic. We could use any of a number of other names:
    - `RegistrationIdentifier`
    - `RegistryIdentifier`
    - `FactoryId`
    - `RegistryId`
    - `RegistrationId`
    - etc.

- Similarly, is `FactoryTypeIdentifier` the correct name? Alternatives might be:
    - `FactoryRegistrationIdentifier`
    - `FactoryTypeRegistrationIdentifier`
    - `FactoryRegistryIdentifier`
    - `FactoryTypeRegistryIdentifier`
    - `FactoryIdentifier` (mutually exclusive with using it as a replacement for `Identifier`
    - `FactoryRegistryId`
    - `FactoryTypeRegistryId`
    - etc.

- Are `identifier` and `type` the right names for the arguments in the new design?

## Appendix: TypeScript

This design is motivated in part by a desire for the API to better support type-safe TypeScript usage. This appendix explains the motivation and design as it impacts the declarations maintained by the Typed Ember team. The big idea here is to make registry lookups type-safe, including support for autocomplete and refactoring, while not making the boilerplate situation *worse* (eliminating it entirely is unfortunately not yet possible).

### TypeScript motivation

The type definitions for Ember currently make use of a pattern we call a “type registry”: a mapping of *strings* to *types*. This allows the types to safely represent string-based lookups like `story.findRecord('some-model')` or `service('session')`.

For example, to make service lookups work with the classic API, users had to write something like this:

```ts
import Service from '@ember/service';

export default class Session extends Service {
  login(email: string, password: string) {
    // ...
  }
  
  logout() {
    // ...
  }
}

declare module '@ember/service' {
  interface Registry {
    session: Session;
  }
}
```

Then, users could write fairly idiomatic Ember and have the types resolve correctly:

```ts
import Component from '@ember/component';
import { inject as service } from '@ember/service';

export default Component.extend({
  session: service('session'), // type: `Session`
})
```

The Typed Ember blueprints generate that boilerplate for users, which (hopefully) minimizes the pain of working with these definitions. However, to make this work for the registry APIs, users (and therefore the blueprints) would have to add *another* declaration to the bottom of the file:

```diff
  import Service from '@ember/service';
  
  export default class Session extends Service {
    login(email: string, password: string) {
      // ...
    }
    
    logout() {
      // ...
    }
  }
  
  declare module '@ember/service' {
    interface Registry {
      session: Session;
    }
  }
  
+ declare module 'TBD-some-registry-spot' {
+   interface Registry {
+     'service:session': typeof Session;
+   }
+ }
```

This *works*, but is additional boilerplate and, more importantly, *duplication of information*, which can get out of sync. Both of these problems are avoided by the new design proposed by this RFC. With the new design, users would still need the same boilerplate required today, but the types for `lookup` and other registry APIs can be extended in a way transparent to users so that they “just work” with *zero* additional effort, as covered in the next section.

### Proposed type definitions

If we were to ship this, we would make one significant change to the *existing* types on DefinitelyTyped, supplying a codemod for ember-cli-typescript users. Where today, the various type registries refer to the class types, we would update them to refer to the *constructor* type for the class. For example, for the service registry:

```diff
 interface Registry {
-  session: Session;
+  session: typeof Session;
 }
```

This single change unlocks the ability to rigorously and exhaustively supply types for the API proposed above, with no further changes to existing user code.

We would introduce the following types at DefinitelyTyped for the new API, along with rigorous type tests to guarantee they provide the guarantees they specify. (A basic working demo of this is available on [this TypeScript playground][types-demo].)

**Note:** the injection types for service and controller are *slightly* simplified here to demonstrate how they work in practice; in fact they will include the appropriate `PropertyDecorator` semantics (as they do today).

```ts
// ----- type utilities ----- //

type ConstructorArgs<T> = T extends new (...args: infer U) => unknown ? U : never;

type InstanceOf<T> = T extends new (...args: any) => infer R ? R : never;

// ----- types to implement this RFC ----- //

// NOTE: some of these lookups will be newly defined as part of this
//       project, e.g. the registries for routes and components.
interface TypeRegistry {
  service: import('@ember/service').Registry;
  controller: import('@ember/controller').Registry;
  route: import('@ember/routing/route').Registry;
  component: import('@typed-ember/component').Registry;
  adapter: import('ember-data/types/registries/adapter').default;
  model: import('ember-data/types/registries/model').default;
  serializer: import('ember-data/types/registries/serializer').default;
  transform: import('ember-data/types/registries/transform').default;
}

interface FactoryIdentifier<
  Type extends keyof TypeRegistry,
  Name extends keyof TypeRegistry[Type]
> {
  type: Type;
  name: Name;
  namespace?: string;
}

interface FactoryTypeIdentifier<Type extends keyof TypeRegistry> {
  type: Type;
}

export interface LookupOptions {
  singleton?: boolean;
  instantiate?: boolean;
  source?: string;
  namespace?: string;
}

interface RegisterOptions {
  singleton?: boolean;
  instantiate?: boolean;
}

interface FactoryManager<Class> {
  class: Class;
  create(
    initialValues?: {
      [K in keyof InstanceOf<Class>]?: InstanceOf<Class>[K];
    }
  ): InstanceOf<Class>;
}

interface Factory<Class> {
  class: Class;
  create(
    initialValues?: {
      [K in keyof InstanceOf<Class>]?: InstanceOf<Class>[K];
    }
  ): InstanceOf<Class>;
}

interface Owner {
  factoryFor<
    Type extends keyof TypeRegistry,
    Name extends keyof TypeRegistry[Type]
  >(
    identifier: FactoryIdentifier<Type, Name>,
    options?: LookupOptions
  ): FactoryManager<TypeRegistry[Type][Name]>;

  hasRegistration<
    Type extends keyof TypeRegistry,
    Name extends keyof TypeRegistry[Type]
  >(
    identifier: FactoryIdentifier<Type, Name>
  ): boolean;

  inject<
    Type extends keyof TypeRegistry,
    Name extends keyof TypeRegistry[Type]
  >(
    factory: FactoryIdentifier<Type, Name> | FactoryTypeIdentifier<Type>,
    property: string,
    injection: FactoryIdentifier<Type, Name>
  ): void;

  lookup<
    Type extends keyof TypeRegistry,
    Name extends keyof TypeRegistry[Type]
  >(
    identifier: FactoryIdentifier<Type, Name>,
    options?: LookupOptions
  ): InstanceOf<TypeRegistry[Type][Name]>;

  register<
    Type extends keyof TypeRegistry,
    Name extends keyof TypeRegistry[Type]
  >(
    identifier: FactoryIdentifier<Type, Name>,
    factory: any,
    options?: RegisterOptions
  ): void;

  registerOptions<
    Type extends keyof TypeRegistry,
    Name extends keyof TypeRegistry[Type]
  >(
    identifier: FactoryIdentifier<Type, Name>,
    options: RegisterOptions
  ): void;

  registerOptionsForType<Type extends keyof TypeRegistry>(
    identifier: FactoryTypeIdentifier<Type>,
    options: RegisterOptions
  ): void;

  registeredOption<
    Type extends keyof TypeRegistry,
    Name extends keyof TypeRegistry[Type],
    OptionName extends keyof RegisterOptions
  >(
    identifier: FactoryIdentifier<Type, Name>,
    optionName: OptionName
  ): Pick<RegisterOptions, OptionName>;

  registeredOptions<
    Type extends keyof TypeRegistry,
    Name extends keyof TypeRegistry[Type]
  >(
    identifier: FactoryIdentifier<Type, Name>
  ): RegisterOptions;

  registeredOptionsForType<Type extends keyof TypeRegistry>(
    type: FactoryTypeIdentifier<Type>
  ): RegisterOptions;

  resolveRegistration<
    Type extends keyof TypeRegistry,
    Name extends keyof TypeRegistry[Type]
  >(
    identifier: FactoryIdentifier<Type, Name>
  ): Factory<TypeRegistry[Type][Name]>;

  unregister<
    Type extends keyof TypeRegistry,
    Name extends keyof TypeRegistry[Type]
  >(
    identifier: FactoryIdentifier<Type, Name>
  ): void;
}
```

[types-demo]: https://www.typescriptlang.org/play/index.html#code/PTAEFpM0BcE8AOBTUBXGBLANhzSDOEU4oIAUGfMqAMID2AdvjAE6oDGMdLAgiwOb4APABUAfKAC8oEaCQAPGEgYATQgyQB3UAAoAdAYCGA-AC5QGBgDMkLUAFUAlFImoGAawZ1NDUAH4HUHMNADdbAG4KKhQASSYYQwZ2JAB5K1EJaVkFJVV1LV0DPWNBc0S4Z0kJSxs7ACV-UAbgpDCWSLIQIihQBBYkcAUMZkt+UH7+YdY4WEQCUE1cAAtQQwQ+uj6MQyVQdiXE-nniEnIySyUWK0Nk0ABlWxCMZLqkSeYWGYBvMlBQfAI+AwjHM0ToVnugOBDEiAF8KBdbNdbvQGKw6FgsLZXu9pqAfn81ggcOwdtDQXNwaAeOsSWTGKj0ZiImR4ec0UibigRHMcVNPvjfv9Hs8kOYHiwni83vy4JE-uxGEysSxzIyWBiVXyPnLWRQuidZshCFwLABbYlIM3KGCwJbDJoAMRo3WgZ0RVy5oEdNy4nxiKhtGCsGFsQiFPOoOWUalA7iQcCpkaQ2umABohQA5QzWuSKGOEeOJiHJ1OfADayYAumQJASjWKZHN5aAGDnG9nrS229b8AguX5zB9RnCERzPbcfZxuHBkwGgyGw8m87lY0Wk7yZTq60LouZk6P2ZdkSgJVKkHEAFZITjQ+do4OhlhCTsoaN5OMJqln0VluA7v4ew7dtu3bPsByHVgRz1TowAUBBuCUFRVkIAADSxr04VC01bdtkNQgFJVFVDQCWWwUCsbg9kYMIGFDJIkDIQN2CwYwKLcW9GGFIjkhfdsVwLT9i0hHiUy3aYxB0IDzFfRxzDiZhEmSNIhB-aVcQrV8qzESJmNY-pQCsDjMC4wjzz43N3zXL8ITUsSNP-HQhQwQMH0XVURPPK8bxMhh70wdyLKQMQyDk0AFISBiVLsv9yy0nSx2PL11U1WxvM4vzXICp8goEj91whFLmRYP8ANw60ZJAoUgPA5JB3+KCGH4Q89LYwzjOhai0Q1YrcqswsbNoJUeq1cTPkk6TQFk+T4iU1J0iK0aHLi9ttN0m99PYpJfK65Uw1fPLrOExbsTGxznKyx9bDVYbUpYdLfP8q7n1fEKwoiuaVJOkqzpW601v1OD5AQlgkJQ0B0IYTCYGw8qkHwxVurukiyIMyi7ER2j6OSCh4MQixxxPUAABk6DodxUAQFIEF8wh6yBJqsS4Bh6oAIzJrFEhbSxFIfHYkDZjmkC5oV8DoVAWDqyCWGgwCwP7KWGplprDw9InU1sanacFP4Gf4JnGEFjFhZhZzZr5pQjc5022TVr0pz9OAAFlEkMI5nxoVj8HwMqWMMb21S9-AW3Yfp+acv4-ksXBtiwAA1QwsFQAh6vrSPQHLABpAmhKpD6ooWoOxCrer8+Uwv-Z9rOqxbP54T+d7zfLoRPcrhLbcJ+3fRnFui51vYg8DyuQ7DpQI8j6PMEThOk5T8w08jrOc4K8Km-m3u25Lmbeeb1vvbEava9AevQEbnf173n3Vc724Uh8Wx++RR3HW4cN0+Xfrc5LTcHIzdODs-ivUsv1qxCkkkKKOl13LmAdjOJ6gVkw4Ven-SOmxab1VJuTSmWtoT4CFGFWBnwXZtndqIH+spKxzCrH9JAAMhQHHwGWekDA36Rw-vmfKg1gG-wgVNfigCuHkJ1JQ5ANY-jgPTi5BcT4YHd39FAnKiC+HWhCg3cw7NjYizNtDVhfx2GrgGsJbhsoUF-AARwo6G5kCxVAeI8efwn4zlkdOeR0ilxzCQe2CQAAfb0cjZxzHgYouYYhTG9A1MgUGcBpajDCRhHy5I-EuLgEE9xyBPEqPweYEIdAXIdD+FgMmFMEC6KbFGCxhirH2RMbw8xBiv5lOqcI2xoAJETwUddJJjtUnPiUcg3haDcEYKKdgmmuCsmr3PipYxzSqE0LoX8CYUwwy8P0YJIBQj0y1P4RUhpMzpgiNoWA+xFgOkeUISks5ZD0nKOCmExxnwygMDgGEwZjB8D1Q1iwHB7yJk5LyRQRZ4lNZjPeaUtZnCjGbM+GEup6zBHWJAVQ45vCpFuRkV0uBVy+leNeaCpg5gvk-KYH83JKh8njGBd8-F+AX4sGTNct8uyNmIocm0yBbjzn+LnNikJeLaaEqpcSvBajQD-PJYCyluJyIqGJeCuYh1Knf1ZTU-+Oz6ksqaQc6sYTiVwshVSIlNKUWSLOc47pvKbn9PTm8hgr5zB6vbBMgACs8dwQgjW0xwo6lRFKlnMBlcK+V5SNUIq1TC7ZllmVhpsciuxqKzWYtceitJSAMnBQmZ63BfqqXw2FXShlELLHKvDeddOe4k0BOQD0xlqjT6CuldS2mOaxZYDCEw3ywamWhqhSqnUsL1Xwt7aWw5YjWknLRdlTpFya04syaKi5jLY2iPmQlIUbh-WXC7YqvZ0KXmRu7UOqpy6jnxtNZy81WLOWMvTXWsK4rDwGmIA2Qg6NQCBjNHQV0pxgAUD9t7SE3tOr1kRh8DgfodBbBCPzUA3gNAeTvvB5wXw2QFLoJMBgOhUCEXtUrWJvRK6aG4CoGJTUwrOo1GaYYSAhDirKkCmAEtfAUboFRgEeh+itrCDoL4aAcPthwv2b2RGWDIVhI4PQMAyKYZ0JUejkdQPGz0IU-gOgABE+AODJG9gAQjU44I+4mWyob47YVOvDsO2Fw8OFWQpYSHn-YQGkxJnjMO+v3VqBlP2BiwPPVswsuDqKFokY+LUNptSZrB++CHouREiwCIDjALlSCi-BvQDy4B0p4w2cwGmRTJDUzhSaeXEsMDU8fAzZBIsPFK5fFLCWgRJf8XoRzcWkC2ga9CAAmkgBAKWNDaBq41hgl8dBwdsJV+LUJGCZk2CgaQA3APDdGwZ0gYAAAkABRYGPlNssA1CwKr7XhSlc22aVmtgsAzGkJ1prySWujyQDx9dhF54Wf4xVUAan2aszU3Z1kEmVOWB0AAch+yDnCIOhP4BEyoEHlWyDjZYMpkZCBssVrU4jPaLBCtw1y5eYwrMsT4HK0ZtboAts7c4Htg7FAkccdzbK-F49eMY6xyNWwuPitEjpL5UnKC1M80ilPJQf2JNC8SCLpAHR6eFKwWj1ncxctmVFFz9syvptlYqyjjD6mfu47U9D2H+nIiy9R+jpXX32d3TV59tTPPXN8+195pAWBTfRYZ1x8NzCLfIFy9b4qtvGz29pI76EpOJOhwC09+sLvfP90AgFugoI2CMTrgDmXHu5fFN942EHAeVQQ7x6AEHDvSS+RBxVunHuN1Utzxr0SQeNelYjx0FXyRQe3YYPDyI7eY-F5B13yvRmyB9+y5NQfmui81QVnn-WdBWaJ2HwjgvthQdl+YT3sgq+WDj-VyXjfFeKuRB33vz7pfQ-l+hNP+WXJzCD9YwMZQGGkDL8iEAA