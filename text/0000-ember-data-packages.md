- Start Date: (fill me in with today's date, 2018-10-31)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Ember Data Packages

## Summary

This documents presents the proposed **public** import path changes for `ember-data`.
 How *exactly* we re-align any internals is informed but not determined by this.

## Motivation

 * Unify import location `import { Model } from 'ember-data'` or `import Model from 'ember-data/model'` ?
 * Drop the concept of a single `DS` namespace
 * Unlock the potential to enable end users to drop unneeded portions of ember-data

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
      <td>DS.RESTAdpter</td>
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
      <td>import EmbeddedRecordsMixin from 'ember-data/serializers/rest/mixins';</td>
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

  3) normalizeModelName is defined... very oddly. Why? Also we should probably deprecate this
   and continue to move to a world in which less normalization of modelName is required.

#### `@ember-data/relationship-layer`

This package seems thin but it's likely to hold quite a bit.
  Additional private things that would be moved here:
  
  * everything in `-private/system/relationships/state`
  * `BelongsToReference` and `HasManyReference`
  * relationship logic from `store` / `internal-model` that need to be isolated and extracted

#### `@ember-data/debug`

  Moving `DebugAdapter` here would allow dropping it if not desired. I already suspect we should
  RFC dropping it for production builds where it adds persistent unnecessary overhead for a tool
  meant for devs. This exists to support the ember inspector.

### Documented Public APIs without public import paths

There are a few public classes that are not exposed at all via `export` today. Those classes will not be given
  public export paths, but the package containing their documentation and implementation is shown here:

  * `@ember-data/store`
    * `Reference`
    * `RecordReference`
  * `@ember-data/relationship-layer`
    * `BelongsToReference`
    * `HasManyReference`
  * `@ember-data/model`
      * `PromiseBelongsTo`
      * `PromiseRecord`

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
