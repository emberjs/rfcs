- Start Date: 2017-07-25
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Align Ember Data module API with [RFC #176](https://github.com/emberjs/rfcs/blob/master/text/0176-javascript-module-api.md).

# Motivation

This document proposes changes to the modules exported by Ember Data to make it consistent with the changes to the Ember module API proposed in [RFC #176](https://github.com/emberjs/rfcs/blob/master/text/0176-javascript-module-api.md).

# Detailed design

Ember Data will stick with 1 top level namespace of `@ember-data`.
Under it there are 6 nested module namespaces where exposed components could live.

- `@ember-data/store` - Store related concers with storing records inside ember data's identity map.
- `@ember-data/model` - Classes and utilities related to modeling data
- `@ember-data/adapter` - Classes and utilities related to communicating with external backends
- `@ember-data/serializer` - Classes for serializing and extracting data from outside Ember Data
- `@ember-data/transform` - Classes for transforming individual values
- `@ember-data/promise-proxies` - Classes for allowing promises to show up in ember templates.

# How We Teach This

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

How should this feature be introduced and taught to existing Ember
users?

# Drawbacks

Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

There are tradeoffs to choosing any path, please attempt to identify them here.

# Alternatives

The easiest alternative would be to continue using the current exports.
Alternatively, we could use the current shims, which do not follow the RFC #176 heuristics as closely.

# Unresolved questions

## Should transforms live inside `ember-data/serializer`?

Conceptually transforms are closly related to serializers,
byt theyhave their own name and namespace in the Ember container.

# Addenda

## Addendum 1 - Table of Module Names and Exports by Global

### `@ember-data/store`
| Global                                | Module                                                                      |
|---                                    | ---  |
| `DS.Store` | `import Store from '@ember-data/store'` |
| `DS.AdapterPopulatedRecordArray` | `import AdapterPopulatedRecordArray from '@ember-data/store/src/adapter-populated-record-array'` |
| `DS.DebugAdapter` | `import DebugAdapter from '@ember-data/store/src/debug-adapter'` |
| `DS.FilteredRecordArray` | `import FilteredRecordArray from '@ember-data/store/src/filterd-record-array'` |
| `DS.ManyArray` | `import ManyArray from '@ember-data/store/src/many-array'` |
| `DS.RecordArray` | `import RecordArray from '@ember-data/store/src/record-array'` |
| `DS.RecordArrayManager` | `import RecordArrayManager from '@ember-data/store/src/record-array-manager'` |
| `DS._initializeStoreService` | `import { initializeStoreService } from '@ember-data/store'` |
| `DS.normalizeModelName` | `import { normalizeModelName } from '@ember-data/store'` |
| `DS._setupContainer` | `import { setupContainer } from '@ember-data/store'`

### `@ember-data/model`
| Global                                | Module                                                                      |
|---                                    | ---  |
| `DS.Model` | `import Model from '@ember-data/model'` |
| `DS.Errors` | `import Errors from '@ember-data/model/src/errors'` |
| `DS.InternalModel` | `import InternalModel from '@ember-data/model/src/internal-model'` |
| `DS.Relationship` | `import Relationship from '@ember-data/model'` |
| `DS.RootState` | `import RootState from '@ember-data/model/src/root-state'` |
| `DS.Shapshot` | `import Snapshot from '@ember-data/model/src/snapshot'` |
| `DS.attr` | `import { attr } from '@ember-data/model'`|
| `DS.belongsTo` | `import { belongsTo } from '@ember-data/model'`|
| `DS.hasMany` | `import { hasMany } from '@ember-data/model'` |


### `@ember-data/adapter`
| Global                                | Module                                                                      |
|---                                    | ---  |
| `DS.Adapter` | `import Adapter from '@ember-data/adapter'` |
| `DS.AbortError` | `import AbortError from '@ember-data/adapter/src/abort-error'` |
| `DS.AdapterError` | `import AdapterError from '@ember-data/adapter/src/adapter-error'` |
| `DS.BuildURLMixin` | `import BuildURLMixin from '@ember-data/adapter/src/build-url-mixin'` |
| `DS.InvalidError` | `import InvalidError from '@ember-data/adapter/src/invalid-error'` |
| `DS.JSONAPIAdapter` | `import JSONAPIAdapter from '@ember-data/adapter/src/json-api'` |
| `DS.RESTAdapter` | `import RESTAdapter from '@ember-data/adapter/src/rest'` |
| `DS.TimeoutError` | `import TimeoutError from '@ember-data/adapter/src/timeout-error'` |
| `DS.errorsHashToArray` | `import { errorsHashToArray } from '@ember-data/adapters'` |
| `DS.errorsArrayToHash` | `import { errorsArrayToHash } from '@ember-data/adapters'` |

### `@ember-data/serializer`
| Global                                | Module                                                                      |
|---                                    | ---  |
| `DS.Serializer` | `import Serializer from '@ember-data/serializer'` |
| `DS.EmbeddedRecordsMixin` | `import EmbeddedRecordsMixin from '@ember-data/serializer/src/embedded-records-mixin'` |
| `DS.JSONAPISerializer` | `import JSONAPISerializer from '@ember-data/serializer/src/json-api'` |
| `DS.JSONSerializer` | `import JSONSerializer from '@ember-data/serializer/src/json'` |
| `DS.RESTSerializer` | `import RESTSerializer from '@ember-data/serializer/src/rest'` |

### `@ember-data/transform`
| Global                                | Module                                                                      |
|---                                    | ---  |
| `DS.Transform` | `import Transform from '@ember-data/transform'` |
| `DS.BooleanTransform` | `import BooleanTransform from '@ember-data/transform/src/boolean'` |
| `DS.DateTransform` | `import DateTransform from '@ember-data/transform/src/date'` |
| `DS.NumberTransform` | `import NumberTransform from '@ember-data/transform/src/number'` |
| `DS.StringTransform` | `import StringTransform from '@ember-data/transform/src/string'` |

### `@ember-data/promise-proxy`
| Global                                | Module                                                                      |
|---                                    | ---  |
| `DS.PromiseArray` | `import PromiseArray from '@ember-data/promise-proxies/src/array'` |
| `DS.PromiseManyArray` | `import PromiseManyArray from '@ember-data/promise-proxies/src/many-array'`|
| `DS.PromiseObject` | `import PromiseObject from '@ember-data/promise-proxies/src/object'` |
