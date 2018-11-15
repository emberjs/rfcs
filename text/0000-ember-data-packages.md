- Start Date: (fill me in with today's date, 2018-10-31)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Ember Data Packages

## Summary

This documents presents the proposed **public** import path changes for `ember-data`, and moving `ember-data`
  into the `@ember-data` namespace.

## Motivation

**Reduce Confusion & Bike Shedding**

Users of `ember-data` have often noted their confusion by the existence of both direct and "god object" (`DS.`) style
 imports for modules from `ember-data`. The documentation currently uses primarily the `DS.` style, and users have
 expressed interest and confusion over why the documentation has not been updated to reflect direct imports.

**Improve The TypeScript Experience**

Presence of multiple import locations confuses `Typescript`'s autocomplete, symbol resolution, and type hinting.
 
**Simplify The Mental Model**

Users of `ember-data` complain about the large API surface area; however, a large portion of this surface area is
 non-essential user-land APIs that the provided adapter and serializer implementations expose. This move to packages
 helps us simplify the mental model in three ways.
 
 First: it gives us a natural way of dividing the documentation and learning story such that key concepts
   and APIs are more discoverable.
 
 Second: it allows us specifically to isolate the API surface area explosion of the provided adapter and serializer
   implementations and make it clear that these are non-essential, replaceable APIs. E.G. it will help us to communicate
   that these adapters and serializers are _an implementation_, **not** _the required implementation_.
   
 Third: it clarifies the roles of several concepts within `ember-data` that are often misused today. Specifically:
   the `embedded-records-mixin` should *_only_* be used with the `RESTAdapter`, and `transforms` are *_only_* a
   serialization/deserialization concern and not a way of defining custom `attrs` or `types`. Furthermore, `transforms`
   are only applicable to the serializer implementations that `ember-data` provides, and not to `custom` (and sometimes
   not to `subclassed`) serializers.

**Improve the Contributor Experience**

Contributors to `ember-data` are faced with a large, complex project with poor code and test organization. This makes it
 unduly difficult to discover what tests exist, where to add tests, where associated code lives, and even what parts of
 the code base relate to the feature or bug that they are looking to address.
 
 This move to packages will help us restructure the project and associated tests in a manner that is more discoverable.

**Provide a Clear Subdivision of Packages**

Today, `ember-data` is a large single package (`~35KB gzipped` in production). `ember-data` is often one of the largest
 dependencies `emberjs` users have in their applications. However, not all users utilize all parts of `ember-data`, and
 some users use very little. Providing these packages helps to clearly show the cost of various features, and better
 allows us to enable end users to eliminate unneeded packages.

Users that implement their own adapter os serializers today must still carry the significant weight of the adapter and
 serializer implementations that `ember-data` ships regardless. This is a weight we should enable these users to eliminate.

With the landing of `RecordData` and the merging of the `modelFactoryFor` RFC, it is likely that many applications
 will soon require far less of `ember-data` than they do today. `ember-m3` is an example of a project that utilizes these
 APIs in a way that requires significantly less of the `ember-data` experience.

**Provide Infrastructure for Additional Changes**

`ember-data` is entering a period of extended evolution, of which `RecordData` and `modelFactoryFor` are only the early
  pieces. For example, current thinking includes the possibility of `ember-data` evolving to provide an `ember-m3`-like
  experience for `json-api` as the default out-of-the-box experience, and a rethinking of how we manage the request/response
  lifecycle when fulfilling a request for data.
  
 These experiences would live alongside the existing experience for a time prior to any deprecations of the current layer,
  and it is possible that sometimes the current experience would never be deprecated. Subdividing `ember-data` into these
  packages will enable us to provide a more seamless transition between these experiences without hoisting any package
  size costs onto users that do not use either the current or the new experience.

**Improve our CI Time**

Currently `ember-data` lives in the `emberjs` organization on `github` despite owning the `ember-data` organization.
  Other core projects (`ember-cli`, `glimmer`, `ember-learn`) have already established the value of a core team utilizing
  their own organization. This includes improvements in managing membership permissions for their respective teams.
  
Today, presence in the `emberjs` organization on `github` also places `ember-data` in the
  `emberjs` organization on `Travis`, where we compete for a shared pool of test instances, often delaying our tests
  by as much as an extra hour. A move to the `ember-data` organization would move our `CI` runs out of the `emberjs`
  on `Travis` into our own organization, removing us from the shared pool. 

## Detailed design

<table>
  <thead>
    <tr>
      <th colspan="2">Before</th>
      <th>After</th>
    </tr>
    <tr>
        <th>import DS from 'ember-data';</th>
        <th>Direct Import</th>
        <th>New Location</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td colspan="3"><h3>@ember-data/model</h3></td>
    </tr>
    <tr>
      <td>DS.Model</td>
      <td>import Model from 'ember-data/model';</td>
      <td>import { Model } from '@ember-data/model';</td>
    </tr>
    <tr>
      <td>DS.attr</td>
      <td>import attr from 'ember-data/attr';</td>
      <td>import { attr } from '@ember-data/model';</td>
    </tr>
    <tr>
      <td>DS.belongsTo</td>
      <td>import { belongsTo } from 'ember-data/relationships';</td>
      <td>import { belongsTo } from '@ember-data/model';</td>
    </tr>
    <tr>
      <td>DS.hasMany</td>
      <td>import { hasMany } from 'ember-data/relationships';</td>
      <td>import { hasMany } from '@ember-data/model';</td>
    </tr>
    <tr>
      <td>DS.Errors</td>
      <td>none</td>
      <td>import Errors from '@ember-data/model/errors';<br>
        <br>+ deprecate directly importing this class
      </td>
    </tr>
    <tr>
      <td>DS.PromiseManyArray</td>
      <td>none</td>
      <td>import PromiseManyArray from '@ember-data/model/promise-many-array';<br>
        <br>+ deprecate directly importing this class
      </td>
    </tr>
    <tr>
      <td>DS.ManyArray</td>
      <td>none</td>
      <td>import ManyArray from '@ember-data/model/many-array';<br>
        <br>+ deprecate directly importing this class
      </td>
    </tr>
    <tr>
      <td>(@private) DS.InternalModel</td>
      <td>none</td>
      <td>none</td>
    </tr>
    <tr>
      <td>(@private) DS.RootState</td>
      <td>none</td>
      <td>none</td>
    </tr>
    <tr>
      <td colspan="3"><h3>@ember-data/adapters</h3></td>
    </tr>
    <tr>
      <td>DS.Adapter</td>
      <td>import Adapter from 'ember-data/adapter';</td>
      <td>import Adapter from '@ember-data/adapters';</td>
    </tr>
    <tr>
      <td>DS.RESTAdapter</td>
      <td>import RESTAdapter from 'ember-data/adapters/rest';</td>
      <td>import RESTAdapter from '@ember-data/adapters/rest';</td>
    </tr>
    <tr>
      <td>DS.JSONAPIAdapter</td>
      <td>import JSONAPIAdapter from 'ember-data/adapters/json-api';</td>
      <td>import from '@ember-data/adapters/json-api';</td>
    </tr>
    <tr>
      <td>DS.BuildURLMixin</td>
      <td>none</td>
      <td>import BuildURLMixin from '@ember-data/adapters/mixins/build-url';</td>
    </tr>
    <tr>
      <td>DS.AdapterError</td>
      <td>import { AdapterError } from 'ember-data/adapters/errors';</td>
      <td>import { AdapterError } from '@ember-data/adapters/errors';</td>
    </tr>
    <tr>
      <td>DS.InvalidError</td>
      <td>import { InvalidError } from 'ember-data/adapters/errors';</td>
      <td>import { InvalidError } from '@ember-data/adapters/errors';</td>
    </tr>
    <tr>
      <td>DS.TimeoutError</td>
      <td>import { TimeoutError } from 'ember-data/adapters/errors';</td>
      <td>import { TimeoutError } from '@ember-data/adapters/errors';</td>
    </tr>
    <tr>
      <td>DS.AbortError</td>
      <td>import { AbortError } from 'ember-data/adapters/errors';</td>
      <td>import { AbortError } from '@ember-data/adapters/errors';</td>
    </tr>
    <tr>
      <td>DS.UnauthorizedError</td>
      <td>import { UnauthorizedError } from 'ember-data/adapters/errors';</td>
      <td>import { UnauthorizedError } from '@ember-data/adapters/errors';</td>
    </tr>
    <tr>
      <td>DS.ForbiddenError</td>
      <td>import { ForbiddenError } from 'ember-data/adapters/errors';</td>
      <td>import { ForbiddenError } from '@ember-data/adapters/errors';</td>
    </tr>
    <tr>
      <td>DS.NotFoundError</td>
      <td>import { NotFoundError } from 'ember-data/adapters/errors';</td>
      <td>import { NotFoundError } from '@ember-data/adapters/errors';</td>
    </tr>
    <tr>
      <td>DS.ConflictError</td>
      <td>import { ConflictError } from 'ember-data/adapters/errors';</td>
      <td>import { ConflictError } from '@ember-data/adapters/errors';</td>
    </tr>
    <tr>
      <td>DS.ServerError</td>
      <td>import { ServerError } from 'ember-data/adapters/errors';</td>
      <td>import { ServerError } from '@ember-data/adapters/errors';</td>
    </tr>
    <tr>
      <td>DS.errorsHashToArray</td>
      <td>none</td>
      <td>import { errorsHashToArray } from '@ember-data/adapters/errors';<br>
         <br>this public method should also be a candidate for deprecation
      </td>
    </tr>
    <tr>
      <td>DS.errorsArrayToHash</td>
      <td>none</td>
      <td>import { errorsArrayToHashr } from '@ember-data/adapters/errors';<br>
        <br>this public method should also be a candidate for deprecation
      </td>
    </tr>
    <tr>
      <td colspan="3"><h3>@ember-data/serializers</h3></td>
    </tr>
    <tr>
      <td>DS.Serializer</td>
      <td>import Serializer from 'ember-data/serializer';</td>
      <td>import Serializer from '@ember-data/serializers';</td>
    </tr>
    <tr>
      <td>DS.JSONSerializer</td>
      <td>import JSONSerializer from 'ember-data/serializers/json';</td>
      <td>import JSONSerializer from '@ember-data/serializers/json';</td>
    </tr>
    <tr>
      <td>DS.RESTSerializer</td>
      <td>import RESTSerializer from 'ember-data/serializers/rest';</td>
      <td>import RESTSerializer from '@ember-data/serializers/rest';</td>
    </tr>
    <tr>
      <td>DS.EmbeddedRecordsMixin</td>
      <td>import EmbeddedRecordsMixin from 'ember-data/serializers/embedded-records-mixin';</td>
      <td>import EmbeddedRecordsMixin from '@ember-data/serializers/rest/mixins/embedded-records';</td>
    </tr>
    <tr>
      <td>DS.JSONAPISerializer</td>
      <td>import JSONAPISerializer from 'ember-data/serializers/json-api';</td>
      <td>import JSONAPISerializer from 'ember-data/serializers/json-api';</td>
    </tr>
    <tr>
      <td>DS.Transform</td>
      <td>import Transform from 'ember-data/transform';</td>
      <td>import Transform from 'ember-data/transforms';</td>
    </tr>
    <tr>
      <td>DS.DateTransform</td>
      <td>import DateTransform from 'ember-data/transforms/date';</td>
      <td>import { DateTransform } from 'ember-data/transforms';</td>
    </tr>
    <tr>
      <td>DS.StringTransform</td>
      <td>import StringTransform from 'ember-data/transforms/string';</td>
      <td>import { StringTransform } from 'ember-data/transforms';</td>
    </tr>
    <tr>
      <td>DS.NumberTransform</td>
      <td>import NumberTransform from 'ember-data/transforms/number';</td>
      <td>import { NumberTransform } from 'ember-data/transforms';</td>
    </tr>
    <tr>
      <td>DS.BooleanTransform</td>
      <td>import BooleanTransform from 'ember-data/transforms/boolean';</td>
      <td>import { BooleanTransform } from 'ember-data/transforms';</td>
    </tr>
    <tr>
      <td colspan="3"><h3>@ember-data/store</h3></td>
    </tr>
    <tr>
      <td>DS.Store</td>
      <td>import Store from 'ember-data/store';</td>
      <td>import Store from '@ember-data/store';</td>
    </tr>
    <tr>
      <td>DS.Snapshot</td>
      <td>none</td>
      <td>none</td>
    </tr>
    <tr>
      <td>DS.PromiseArray</td>
      <td>none</td>
      <td>none</td>
    </tr>
    <tr>
      <td>DS.PromiseObject</td>
      <td>none</td>
      <td>none</td>
    </tr>
    <tr>
      <td>DS.RecordArray</td>
      <td>none</td>
      <td>none</td>
    </tr>
    <tr>
      <td>DS.AdapterPopulatedRecordArray</td>
      <td>none</td>
      <td>none</td>
    </tr>
    <tr>
      <td>DS.RecordarrayManager</td>
      <td>none</td>
      <td>none</td>
    </tr>
    <tr>
      <td>DS._initializeStoreService</td>
      <td>import initializeStoreService from 'ember-data/intialize-store-service';</td>
      <td>none</td>
    </tr>
    <tr>
      <td>DS._setupContainer</td>
      <td>import setupContainer from 'ember-data/setup-container';</td>
      <td>none</td>
    </tr>
    <tr>
      <td>DS.normalizeModelName</td>
      <td>none</td>
      <td>import { normalizeModelName } from 'ember-data/store';<br>
        <br>this public method should be a candidate for deprecation
      </td>
    </tr>
    <tr>
      <td colspan="3"><h3>@ember-data/record-data</h3></td>
    </tr>
    <tr>
      <td>none</td>
      <td>import { RecordData } from 'ember-data/-private';</td>
      <td>import RecordData from '@ember-data/record-data';</td>
    </tr>
    <tr>
      <td colspan="3"><h3>@ember-data/relationship-layer</h3></td>
    </tr>
    <tr>
      <td>DS.Relationship</td>
      <td>none</td>
      <td>none</td>
    </tr>
    <tr>
      <td colspan="3"><h3>@ember-data/debug</h3></td>
    </tr>
    <tr>
      <td>DS.DebugAdapter</td>
      <td>none</td>
      <td>none</td>
    </tr>
  </tbody>
</table>

### Notes

#### `@ember-data/model`
  
  1) `InternalModel` and `RootState` are tightly coupled to the store and to our provided `Model`
    implementation. Overtime we need to uncouple this, but given their coupling to `Model` and our
    desire to enable them to be eliminated from projects not using `Model`, I believe these exports
    belong in `@ember-data/model` for this reason.

  2) Do the following belong in `@ember-data/model`, or in `@ember-data/relationship-layer`?
    if not with relationships, does the package name "relationship-layer" become confusing?
    
   The argument for them being here is they are far more related to the current `Model` ui
    presentation layer than they are to the state book-keeping which could be used with
    alternative implementations.

  * `belongsTo`
  * `hasMany`
  * `PromiseManyArray`
  * `ManyArray`
  
#### `@ember-data/serializers`

  1) We should move automatic registration of transforms into a more traditional
    `app/` directory re-export for the package so that when the package is dropped they
    cleanly drop as well.

#### `@ember-data/store`

  1) _setupContainer registers various adapters and serializers for fallbacks.
  Either we need to deprecate this behavior (preferred), or separate out initialization
  by package, or both. It also eagerly injects store, which we should deprecate but
  can't until there is a `defaultStore` RFC for ember itself.

  2) _initializeStoreService eagerly instantiates the store to ensure that `defaultStore` is our store.
  we should get rid of this but can't until there is a `defaultStore` RFC for ember itself.

  3) normalizeModelName is defined... very oddly. Why? We should probably deprecate this
   and continue to move to a world in which less normalization of modelName is required.

#### `@ember-data/relationship-layer`

This package seems thin but it's likely to hold quite a bit.
  Additional private things that would be moved here:
  
  * everything in `-private/system/relationships/state`
  * `BelongsToReference` and `HasManyReference`
  * relationship logic from `store` / `internal-model` that need to be isolated and extracted

#### `@ember-data/debug`

  Moving `DebugAdapter` here would allow dropping it if not desired. Additionally we should likely
  RFC dropping it for production builds where it adds persistent unnecessary overhead for a tool
  meant for devs. This exists to support the ember inspector.

### Documented Public APIs without public import paths

There are a few public classes that are not exposed at all via `export` today. Those classes will not be given
  public export paths, but the package containing their documentation and implementation is shown here:

  * `@ember-data/store`
    * `Reference`
    * `RecordReference`
    * `StoreWrapper`
  * `@ember-data/relationship-layer`
    * `BelongsToReference`
    * `HasManyReference`
  * `@ember-data/model`
      * `PromiseBelongsTo`
      * `PromiseRecord`

## How we teach this

This RFC should be seen as a continuation of the `javascript-modules` RFC that defined explicit import paths for `emberjs`.

Codemods and lint rules would be provided to convert existing imports to the new syntax. Existing import locations
 would continue to exist for a time but would print build-time deprecations.

Ember documentation and guides would be updated to reflect these new import paths as well as to utilize the new package
 divisions to improve the teaching story.

## Drawbacks

* A Tiny amount of churn

## Alternatives

* Divide into packages without exposing the new division publically
* Don't divide into packages until nebulous future RFCs have landed
* Use the `@ember` namespace.
* Use the `@ember-data` namespace without moving to the `@ember-data` org.
