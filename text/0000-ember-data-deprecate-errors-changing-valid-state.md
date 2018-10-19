- Start Date: 2018-10-04
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Ember Data | Deprecate Errors changing record-state

## Summary

Deprecates the `add` `remove` and `clear` methods of `DS.Errors` in favor of `addErrors`
 `removeErrors` and `clearErrors`.

## Motivation

In `ember-data`, adding, removing, or clearing errors from a record's `errors` object
 will affect the validation state of the record. Previously a [soft-deprecation](https://github.com/emberjs/data/pull/3853) of
 this behavior was added; however, it provided no clear path to resolution. 

By deprecating these methods entirely, and providing replacement methods that do not
 affect the state of the `record`, we provide end users a clear path and a clear timeline
 for resolving the deprecation, and a mechanism through which users can determine if this
 change in behavior requires any additional changes in their application code.

## Detailed design

The warnings in `DS.Errors` `add` `remove` and `clear` will be upgraded to deprecations
 targeting the next LTS `3.8`. This means the feature and these methods will be removed
 in `3.9`.

New methods will be added to `DS.Errors` that do not affect record state. `addErrors` `removeErrors`
 and `clearErrors`. These methods will have the same signatures as before, listed below, where
 `member` is one of `type` `id` or the property name of an `attribute` or `relationship`.

```typescript
interface Errors {
  addErrors(member: string, messages: string|Array<string>): void;
  removeErrors(member: string): void;
  clearErrors(): void;
}
```

## How we teach this

`DS.Errors` is not well documented today, so this is an opportunity for us to revisit and
 clean up.

## Drawbacks

- For apps using `Errors` that do not rely on the changes to record state, this will feel like
 unnecessary churn.
- The new method names may feel unnecessarily verbose

## Alternatives

Two alternatives were considered:

1) upgrading the existing warning to specify that the behavior would be removed in `3.9`,
   We felt this was equally un-actionable as the state of things today.
2) treating the warning as a fully complete deprecation, and removing the undesired behavior now,
   as the warning has been in place long enough to be considered "complete" post the `3.4 lts` cycle.
   We felt this was unfair to consumers who have had no way to act upon the deprecation or determine
   if it would have any effects on their applications.

## Unresolved questions

None
