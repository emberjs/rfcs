---
Start Date: 2018-10-31
Relevant Team(s): Ember Data
RFC PR: https://github.com/emberjs/rfcs/pull/395
Tracking: https://github.com/emberjs/rfc-tracking/issues/11

---

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

Users that implement their own adapter or serializers today must still carry the significant weight of the adapter and
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

## Detailed design

This RFC proposes import paths following the guidelines established in [Ember Modules RFC #176](https://github.com/emberjs/rfcs/pull/176),
  with two addendums to account for scenarios that weren't faced by `ember`:
* `Error` sub-classes are named exports
* `Mixins` are named exports

This is done to allow for continued grouping by common usage and mental model, where otherwise users would be faced with multiple imports from length file paths.

The following modules would continue to live in a monorepo that (until further RFC) would continue to live at `github.com/ember/data`.

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
      <td>import Model from '@ember-data/model';</td>
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
      <td colspan="3"><h3>@ember-data/adapter</h3></td>
    </tr>
    <tr>
      <td>DS.Adapter</td>
      <td>import Adapter from 'ember-data/adapter';</td>
      <td>import Adapter from '@ember-data/adapter';</td>
    </tr>
    <tr>
      <td>DS.RESTAdapter</td>
      <td>import RESTAdapter from 'ember-data/adapters/rest';</td>
      <td>import RESTAdapter from '@ember-data/adapter/rest';</td>
    </tr>
    <tr>
      <td>DS.JSONAPIAdapter</td>
      <td>import JSONAPIAdapter from 'ember-data/adapters/json-api';</td>
      <td>import JSONAPIAdapter from '@ember-data/adapter/json-api';</td>
    </tr>
    <tr>
      <td>DS.BuildURLMixin</td>
      <td>none</td>
      <td>import { BuildURLMixin } from '@ember-data/adapter';</td>
    </tr>
    <tr>
      <td>DS.AdapterError</td>
      <td>import { AdapterError } from 'ember-data/adapters/errors';</td>
      <td>import AdapterError from '@ember-data/adapter/error';</td>
    </tr>
    <tr>
      <td>DS.InvalidError</td>
      <td>import { InvalidError } from 'ember-data/adapters/errors';</td>
      <td>import { InvalidError } from '@ember-data/adapter/error';</td>
    </tr>
    <tr>
      <td>DS.TimeoutError</td>
      <td>import { TimeoutError } from 'ember-data/adapters/errors';</td>
      <td>import { TimeoutError } from '@ember-data/adapter/error';</td>
    </tr>
    <tr>
      <td>DS.AbortError</td>
      <td>import { AbortError } from 'ember-data/adapters/errors';</td>
      <td>import { AbortError } from '@ember-data/adapter/error';</td>
    </tr>
    <tr>
      <td>DS.UnauthorizedError</td>
      <td>import { UnauthorizedError } from 'ember-data/adapters/errors';</td>
      <td>import { UnauthorizedError } from '@ember-data/adapter/error';</td>
    </tr>
    <tr>
      <td>DS.ForbiddenError</td>
      <td>import { ForbiddenError } from 'ember-data/adapters/errors';</td>
      <td>import { ForbiddenError } from '@ember-data/adapter/error';</td>
    </tr>
    <tr>
      <td>DS.NotFoundError</td>
      <td>import { NotFoundError } from 'ember-data/adapters/errors';</td>
      <td>import { NotFoundError } from '@ember-data/adapter/error';</td>
    </tr>
    <tr>
      <td>DS.ConflictError</td>
      <td>import { ConflictError } from 'ember-data/adapters/errors';</td>
      <td>import { ConflictError } from '@ember-data/adapter/error';</td>
    </tr>
    <tr>
      <td>DS.ServerError</td>
      <td>import { ServerError } from 'ember-data/adapters/errors';</td>
      <td>import { ServerError } from '@ember-data/adapter/error';</td>
    </tr>
    <tr>
      <td>DS.errorsHashToArray</td>
      <td>none</td>
      <td>import { errorsHashToArray } from '@ember-data/adapter/error';<br>
         <br>this public method should also be a candidate for deprecation
      </td>
    </tr>
    <tr>
      <td>DS.errorsArrayToHash</td>
      <td>none</td>
      <td>import { errorsArrayToHash } from '@ember-data/adapter/error';<br>
        <br>this public method should also be a candidate for deprecation
      </td>
    </tr>
    <tr>
      <td colspan="3"><h3>@ember-data/serializer</h3></td>
    </tr>
    <tr>
      <td>DS.Serializer</td>
      <td>import Serializer from 'ember-data/serializer';</td>
      <td>import Serializer from '@ember-data/serializer';</td>
    </tr>
    <tr>
      <td>DS.JSONSerializer</td>
      <td>import JSONSerializer from 'ember-data/serializers/json';</td>
      <td>import JSONSerializer from '@ember-data/serializer/json';</td>
    </tr>
    <tr>
      <td>DS.RESTSerializer</td>
      <td>import RESTSerializer from 'ember-data/serializers/rest';</td>
      <td>import RESTSerializer from '@ember-data/serializer/rest';</td>
    </tr>
    <tr>
      <td>DS.JSONAPISerializer</td>
      <td>import JSONAPISerializer from 'ember-data/serializers/json-api';</td>
      <td>import JSONAPISerializer from '@ember-data/serializer/json-api';</td>
    </tr>
    <tr>
      <td>DS.EmbeddedRecordsMixin</td>
      <td>import EmbeddedRecordsMixin from 'ember-data/serializers/embedded-records-mixin';</td>
      <td>import { EmbeddedRecordsMixin } from '@ember-data/serializer/rest';</td>
    </tr>
    <tr>
      <td>DS.Transform</td>
      <td>import Transform from 'ember-data/transform';</td>
      <td>import Transform from '@ember-data/serializer/transform';</td>
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
      <td>DS.normalizeModelName</td>
      <td>none</td>
      <td>import { normalizeModelName } from '@ember-data/store';<br>
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
    implementation. Over time we need to uncouple this, but given their coupling to `Model` and our
    desire to enable them to be eliminated from projects not using `Model`, these concepts belong in `@ember-data/model`, although they will not be given direct import paths.

  2) The following belong in `@ember-data/model` and not in `@ember-data/relationship-layer` with
    relationships.  While this presents a mild risk of confusion due to the presence of the
    `relationship-layer` package, the argument for their presence here is they are a ui-layer concern being coupled to the current `Model` presentation layer and not related to overall state management
     of relationships which could itself be used with alternative implementations.

  * `belongsTo`
  * `hasMany`

 3) The following have the same considerations as #2 but they will not be given direct import paths.

  * `PromiseManyArray`
  * `ManyArray`

#### `@ember-data/serializers`

  1) We should move automatic registration of transforms into a more traditional
    `app/` directory re-export for the package so that when the package is dropped they
    cleanly drop as well.

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

## Migration

Blueprints, guides, docs, and twiddle would be updated to use the new `@ember-data/` package imports.

A codemod would be provided to convert from the existing import locations to the new ones, as well as lint rules for encouraging their use.

The package `ember-data` would continue to exist, much like `ember-source`. Initially, this package would provide all of the subpackages
 as dependencies as well as the respective re-exports for supporting the existing import paths. After a time, the existing paths would
 be deprecated.

Users who have resolved the deprecations may choose to convert to consuming only the packages they still require directly,
 by dropping `ember-data` from their `package.json` and adding in the individual `@ember-data/` packages as necessary.

Ultimately, the default `ember-data` story in `ember-cli` would change to install select packages from `@ember-data` directly.

## How we teach this

This RFC should be seen as a continuation of the `javascript-modules` RFC that defined explicit import paths for `emberjs`.

Codemods and lint rules would be provided to convert existing imports to the new syntax. Existing import locations
 would continue to exist for a time but would at some point in the future be made to print build-time deprecations.

End users would need to run the codemod at some point, but no other changes will be required.

Ember documentation and guides would be updated to reflect these new import paths as well as to utilize the new package
 divisions to improve the teaching story.

## Drawbacks

* A Tiny amount of churn
* Sub-packages will require sprinkling significant numbers of excess package.json files throughout our repo.
* Our import paths may not align with the expected mental model for addon import paths going forward (no `/src/` in path)

## Alternatives

1) Divide into packages without exposing the new division publicly

  * _argument for:_ Don't expose churn to end users without a clear win, we aren't 100% sure what belongs in a vague
    "future ember-data", so wait until we are sure.
  * _rebuttal:_ The churn is minimal and mostly automated (codemod). There are clear wins here for many users. We
    should not hold up progress now on an uncertain future. Dividing into packages now gives us more options for how to
    manage future evolution. Regardless of when we become certain of what belongs in "future ember-data", these packages
    would need to exist alongside at least for a time.

2) Don't divide into packages until nebulous future RFCs have landed

  * _argument for:_ This argument is an extension of _alternative 1_ in which we wait for specific concepts to mature and
    materialize that we have discussed internally, including a significant rework of how we manage the `request/response`
    lifecycle. These new feature RFCs would come with corresponding deprecation RFCs for parts of the system they either
    fully replace or make vestigial.
  * _rebuttal:_ The argument here is a variation of the argument in _alternative 1_ and the rebuttal merely extends
    that rebuttal as well. These future deprecations would necessarily be long-tail, if we deprecate at all. There is
    the option to have both old and new experiences live side-by-side. Additionally, if we deprecate and then land
    `@ember-data/packages` there is both an equal amount of churn and fewer options for how to manage those deprecations.

3) Use the `@ember` namespace.

  * _argument for:_ `ember-data` is an official package and we wish to position it centrally within the `ember`
    ecosystem. This [argument has been presented](https://github.com/emberjs/rfcs/pull/238#issuecomment-318745236)
    by other core teams in response to previous attempts to move forward with a packages RFC for `ember-data`.
  * _rebuttal:_ `ember-cli` and `glimmer` are also official packages, but with their own namespaces. Additionally
    re-using the `@ember` namespace would only further confusion that many folks already have regarding:
      * where `ember` ends and `ember-data` begins.
      * whether `ember-data` is required or optional
      * whether other data layers are seen as "bad practices" (they are not)
      * what packages are provided by `ember-data` vs `ember`
    `ember-data`'s status as a team, in the guides and in release blog posts on `emberjs.com`, as well as presence in
     the default blueprint provided by `ember-cli` make clear it's status as an official offering. Using the `@ember`
     namespace is not required for this.

    This argument also necessarily foments an untrue presupposition: that `ember-data` is the right choice for every app.
    While we strive to make this the case, it would be very difficult to claim this today, and may never be true,
    as every app presents unique concerns and needs.

    Finally, using the `@ember` namespace would leave us in the unfortunate position of either always scoping all of our
    packages to `@ember/data/` or of fighting with `emberjs` for package names.

4) This RFC but with Adapters and Serializers broken out into the packages `@ember-data/json` `@ember-data/rest` `@ember-data/json-api`.

* _argument for:_ grouping the adapter / serializer "by API spec" feels more natural and would allow for users to drop only the versions of adapters / serializer they don't require.

* _rebuttal:_ Even without considering future changes to `ember-data`'s API surface, there are several issues with this approach.

  1) The implementations inherit each other:
     * `JSONAPISerializer extends RESTSerializer extends JSONSerializer extends Serializer`
     * `JSONAPIAdapter extends RESTAdapter extends Adapter`
  2) The adapter / serializer pairings aren't coupled
     * It is fairly common to use the `JSONAPIAdapter` with the `RESTSerializer` or
    with a custom serializer that extends the `RESTSerializer` and vice-verse.
     * Even when using a consistent spec (`json-api` or `rest`) it is common to need
    a fully custom serializer. The division of needs is at least equally between
    adapter/serializer as it is between specs.

  3) Transforms are an implementation detail for all the provided serializers

     * But they  are not required and likely not even used by custom serializers.

  4) Packages for automatically registered fallbacks would fit poorly.
     * Serializers: `"-default"` `"-rest"` `"-json-api"`
     * Adapters: `"-rest"` `"-json-api"`
  5) Today, we use multiple serializers for a single type based on entry-point
     * `Model.serialize` (per-type) / `Model.toJSON` (`"-json"`) / `Adapter.serialize` (per-adapter)

  That said, this organization is also one of the only-nods
  to future RFCs this RFC concedes. The existing provided implementations all follow roughly the same interface for their implementations, and that interface is something we strongly wish to change. For this reason, it seems advantageous to keep the existing implementations together such that the delineation between a new experience and this experience can be kept clear.
