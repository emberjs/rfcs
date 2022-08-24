---
stage: accepted
start-date: 2022-08-18
release-date: Unreleased
release-versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
teams:
  - data
prs:
  accepted: https://github.com/emberjs/rfcs/pull/846

---

<!--- 
Directions for above: 

Stage: Leave as is
Start Date: Fill in with today's date, YYYY-MM-DD
Release Date: Leave as is
Release Versions: Leave as is
Relevant Team(s): Fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies
RFC PR: Fill this in with the URL for the Proposal RFC PR
-->

# EmberData | Deprecate Proxies

## Summary

Deprecates usage of proxy apis provided by `Ember.ObjectProxy`, `Ember.PromiseProxyMixin` and `Ember.ArrayProxy` in favor of Promises, Native Arrays, and Native Proxies where appropriate.

Specifically this affects `RecordArray` `AdapterPopulatedRecordArray`, `ManyArray`, `PromiseArray`, and `PromiseRecord` but not `PromiseBelongsTo`.

## Motivation

Historically EmberData built many presentation classes overtop of Framework primitives out of necessity. Native Proxies did not yet exist, Promises were relatively unheard of, and async state management was the Wild West.

Today we have better native primitives available to us, affording us the opportunity to significantly simplify these presentation objects and better align them with user expectations in the modern Javascript world and Octane Ember.

Importantly, this simplification will allow for us to address the performance of the most expensive costs of managing and presenting data. It will also sever one of the last entanglements the core of EmberData has with the Framework. While this RFC does not in itself enable Ember-less usage of EmberData, it does
in effect make this a near possibility. 

## Transition Path

### PromiseRecord|PromiseArray

`id: 'ember-data:deprecate-promise-proxies'`

> Accessing any value on the proxy that is not a promise method will trigger a deprecation. This is similar in effect to [RFC 795](https://rfcs.emberjs.com/id/0795-ember-data-return-promise-save). Additionally we deprecate promise state flags, this includes deprecating the flags kept in RFC 795.

**Affected APIs:** `store.findRecord`, `store.queryRecord`, `model.save`, `store.findAll` and `store.query`

**Resolution:**

`await` the returned promise to receive the value to interact with. For promise state flags consider using [ember-promise-helpers](https://github.com/fivetanley/ember-promise-helpers)


----

### RecordArray|AdapterPopulatedRecordArray|ManyArray

`id: 'ember-data:deprecate-ember-array-like'`

> Utilizing EmberObject APIs or ArrayLike APIs that do not correlate to Native Array APIs will be deprecated.

- **Affected APIs:** `store.query|findAll|peekAll` and `hasMany` relationships on `@ember-data/model`
- **Resolution:** Convert to the corresponding native array API. In the most common case of `toArray()` this means to use `slice()` instead.

----

### PromiseBelongsTo

Since there is no path for a PromiseProxy to not exist for belongsTo relationships, deprecating 
this promise proxy is left for the deprecation of Model / belongsTo more broadly.

- **Affected APIs:** async `belongsTo` relationships on instances of `@ember-data/model`

In the interest of parity and in order to make native property access usage easier to refactor to we considered converting PromiseBelongsTo into a native proxy which would allow dot notation access to work. However, this would encourage not resolving the value before interacting with it, and encouraging folks to refactor towards `await` before use is key for the next stage in which async relationships will not exist at all in their current form. For this reason, we choose to leave this Proxy as-is.

We would note that even if Model / belongsTo are not deprecated before 5.0, their replacements will be available.

## How We Teach This

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

No, the current guides do not make use of proxy behavior.

> Does it mean we need to put effort into highlighting the replacement
functionality more? What should we do about documentation, in the guides
related to this feature?  

We should likely create examples of loading and working with async data within components, as it is not always the case that users resolve data in a parent component or route before passing it down. Generally this means surfacing patterns around `resources` and async template helpers like `ember-promise-helpers`.

> How should this deprecation be introduced and explained to existing Ember
users?

The APIs were necessary in the classic world, but not in the Octane world. This simplification will not only allow for us to reduce the size, complexity and performance costs of these classes while also aligning their usage with the "just Javascript" mental model.

## Drawbacks

As with all deprecations, apps will experience some churn. For apps using EmberData, the churn in the 4.x cycle is likely to be high, of which this will be yet-another contributing factor.

## Alternatives

The alternative (not doing this) would prevent deprecation and removal of classic/legacy ember framework code more broadly, as well as it would prevent EmberData from progressing towards being decoupled from the framework and eliminating performance bottlenecks. The cost of this legacy infrastructure is as high as 30% of the overall runtime cost of EmberData.
