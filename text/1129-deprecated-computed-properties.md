---
stage: accepted
start-date: 2025-08-12T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
  - learning
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1129
project-link:
---

# Deprecate Computed Properties and Macros (`@ember/object/computed`, `@ember/object/compat`)

## Summary

Deprecate all remaining classic Computed Property APIs and macros, specifically every export from `@ember/object/computed` and `@ember/object/compat`, in favor of native getters, tracked state, and standard JavaScript utilities or lightweight addon helpers. This removes the CP macro layer (`alias`, `and`, `or`, collection transforms, comparison helpers, etc.) and the interop decorator `dependentKeyCompat`.

## Motivation

Classic computed properties (CPs) and their associated macro helpers predate autotracking and `@tracked`. They provided declarative, dependency-key driven derived state and array/object transforms. Since Ember Octane introduced autotracking and idiomatic use of native getters plus `@tracked`, the macro layer has become:

- Redundant: All use cases can be expressed with native JS, `@tracked`, getters, and small helper functions.
- Costly in bundle size and runtime complexity (dependency key parsing, meta layers, descriptor objects, caching infra, deprecation suppression paths, TS types maintenance).
- A barrier to new learners who must distinguish between classic CPs, tracked properties, and functions/macros.
- An obstacle to long term simplification (removal unlocks internal cleanups in property change events, meta, descriptor caches, legacy invalidation pathways, interop shims for autotracking, and types).

Deprecating these APIs aligns Ember with modern JS ergonomics, lowers cognitive load, and allows focusing docs, tooling, and teaching around a single reactivity model.

## Transition Path

### Scope of Deprecation

All exports from:

- `@ember/object/computed`
- `@ember/object/compat` (only `dependentKeyCompat`)

List of macros slated for deprecation (complete):
`alias`, `and`, `bool`, `collect`, `deprecatingAlias`, `empty`, `equal`, `filter`, `filterBy`, `filterProperty`, `gt`, `gte`, `intersect`, `lt`, `lte`, `map`, `mapBy`, `mapProperty`, `match`, `max`, `min`, `none`, `not`, `notEmpty`, `oneWay`, `or`, `readOnly`, `reads`, `setDiff`, `sort`, `sum`, `union`, `uniq`.

Also implicitly covered: use of `computed()` with dependency key arguments (classic pattern) and setter forms, plus `dependentKeyCompat` decorator.

Out of scope (NOT deprecated by this RFC):
- `@tracked` (unchanged)
- Native getters / setter semantics
- Class field / decorator proposals unrelated to CPs

### General Migration Principles

1. Replace `computed` macro-derived properties with native getters using tracked dependencies.
2. Use standard JS methods (`Array.prototype.filter`, `map`, `some`, `every`, `includes`, spread syntax, `Set`, etc.) or minimal utility functions you write locally.
3. Prefer explicit, readable logic rather than dense macro chains.
4. For aliasing / indirection, refactor to simple getter forwarding or direct property access injection.
5. For one-way / read-only semantics, rely on conventions, TypeScript readonly typings, or simple getters without setters.
6. For mutable derivations once provided by macros that returned array proxies, compute on access within a getter or precompute and cache in a tracked property if performance critical.

### General `computed()` Migration

Classic signature examples slated for deprecation:

```js
fullName: computed('firstName', 'lastName', function () {
  return `${this.firstName} ${this.lastName}`;
}),

hasErrors: computed('errors.[]', function () { /* ... */ }),

sorted: computed('items.[]', 'sortDefinition.[]', function() { /* ... */ }),
```

Migration uses a native getter; dependencies become `@tracked` on the fields accessed.

```js
import { tracked } from '@glimmer/tracking';

class UserModel {
  @tracked firstName;
  @tracked lastName;

  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

Collection dependency keys like `items.[]`, `items.@each.name`, etc. are replaced by direct property access within the getter. Autotracking observes every `@tracked` read. If you push into an array (mutating method) you must reassign to trigger, e.g. `this.items = [...this.items, newItem]`. Codemod will wrap mutating calls or emit TODO comments.

For expensive computations previously cached by computed's default memoization, you may cache manually:

```js
get expensiveResult() {
  if (this._cachedExpensive === undefined || this._cacheKey !== this.inputVersion) {
    this._cachedExpensive = doWork(this.inputs);
    this._cacheKey = this.inputVersion;
  }
  return this._cachedExpensive;
}
```

In practice, most computed properties are inexpensive and can just be recomputed each access.

### Macro-by-Macro Replacement Patterns

Below each macro lists: Purpose, Before, After (tracked + JS), and Codemod Strategy.

#### alias / reads / readOnly / oneWay / deprecatingAlias

Purpose: Indirection or partial mutability semantics.

Before:
```js
import { alias, readOnly, oneWay, reads, deprecatingAlias } from '@ember/object/computed';
fullName: alias('user.fullName'),
userName: reads('user.name'),
title: readOnly('model.title'),
tempName: oneWay('user.name'),
legacyName: deprecatingAlias('user.name')
```
After:
```js
get fullName() { return this.user.fullName; }
set fullName(value) { this.user.fullName = value; }
get userName() { return this.user.name; }
get title() { return this.model.title; } // no setter supplied => read-only
get tempName() { return this._tempName ??= this.user.name; }
set tempName(v) { this._tempName = v; }
// legacyName: delete or treat same as fullName depending on intent
```
Codemod: Replace alias/readOnly/reads with getter. One-way requires backing slot + getter/setter. deprecatingAlias becomes alias + inline deprecation removal.

#### and / or / not / bool / none / notEmpty / empty / equal / gt / gte / lt / lte / match / min / max

Purpose: Boolean / comparison helpers.

After pattern: Direct boolean expression in getter.

Examples:
```js
isReady: and('isLoaded', 'isValid') -> get isReady() { return this.isLoaded && this.isValid; }
isSpecial: or('isAdmin', 'isOwner') -> get isSpecial() { return this.isAdmin || this.isOwner; }
isMissing: none('value') -> get isMissing() { return this.value === null || this.value === undefined; }
hasValue: notEmpty('items') -> get hasValue() { return this.items.length > 0; }
scoreOK: gte('score', 70) -> get scoreOK() { return this.score >= 70; }
label: bool('maybeThing') -> get label() { return !!this.maybeThing; }
fits: match('name', /^user-/) -> get fits() { return /^user-/.test(this.name); }
```
Codemod: Inline replacements building boolean expression; chain of `and`/`or` flattens; `not` wraps `!`; numeric comparators changed to operators; `equal` to `===` (or `==` if historically relied on coercion—guide will recommend strict equality).

#### collect

Before: `items: collect('a', 'b', 'c')`

After: `get items() { return [this.a, this.b, this.c]; }`

Codemod: Getter returning array literal of referenced props.

#### sum / min / max

Before: `total: sum('amounts')`

After: `get total() { return this.amounts.reduce((a,b)=>a+b,0); }`

min/max similarly use `Math.min(...arr)` or reduce.

Codemod: Identify base array path; generate reduce/Math.* pattern.

#### map / mapBy / mapProperty

Before: `names: mapBy('people', 'name')`

After: `get names() { return this.people.map(p => p.name); }`

`map` with callback string transforms similarly; `mapProperty` was legacy alias—same replacement.

Codemod: Replace with `.map` arrow function.

#### filter / filterBy / filterProperty

Before: `enabled: filterBy('features', 'enabled', true)`

After: `get enabled() { return this.features.filter(f => f.enabled === true); }`

`filter` with function path becomes an inline arrow or imported predicate.

Codemod: Build arrow matching parameters; use strict equality when third arg provided.

#### intersect / union / uniq / setDiff

Use native `Set` operations.

Examples:
```js
get shared() { return this.listA.filter(x => this.listB.includes(x)); }
get combined() { return [...new Set([...this.listA, ...this.listB])]; }
get uniqueA() { return [...new Set(this.listA)]; }
get leftOnly() { const b = new Set(this.listB); return this.listA.filter(x => !b.has(x)); }
```
Codemod: Generate patterns; may leave TODO if complex chained macros.

#### sort

Before:
```js
sorted: sort('items', 'sortProps')
sortProps = Object.freeze(['lastName', 'firstName:desc']);
```

After (basic):
```js
get sorted() {
  return [...this.items].sort((a,b) => compareByProps(a,b,this.sortProps));
}
```
Helper (local):
```js
function compareByProps(a,b,props){
  for (let rule of props){
    let [key, dir='asc'] = rule.split(':');
    if (a[key] === b[key]) continue;
    return (a[key] > b[key] ? 1 : -1) * (dir === 'desc' ? -1 : 1);
  }
  return 0;
}
```
Codemod: Inline simple single-key sorts; multi-key yields helper + TODO comment.

#### aliasing of nested arrays (reads etc.)
Covered by the alias section—straight property forwarding.

#### oneWay semantics nuance
`oneWay` copies the value once then decouples. Replacement pattern uses backing slot as demonstrated; assignment inside getter avoided—copy occurs in constructor or first setter invocation depending on semantics needed.

#### readOnly semantics nuance
Simply omit setter. (TypeScript: mark as `get prop(): Type`.)

### Removing `dependentKeyCompat`

This decorator allowed getters that weren't properly autotracked (e.g. using mutation without reassignment internally) to still participate in classic CP invalidation when referenced by classic CPs. With classic CPs removed, there is no remaining need. Remove the import and decorator:

Before:
```js
import { dependentKeyCompat } from '@ember/object/compat';
class Person {
  @dependentKeyCompat
  get legacyFullName() { return this._first + ' ' + this._last; }
}
```
After:
```js
class Person {
  get legacyFullName() { return `${this._first} ${this._last}`; }
}
```
If the getter relied on mutation without reassignment (e.g. pushing into an array it returns), refactor to reassign or compute a derived copy.

### Deprecation Phasing

Suggested timeline (illustrative):
1. Minor (N): Add runtime deprecation warnings for first access/definition of macros and `computed()` with deps; ship codemod & lint rules.
2. Minor (N+1): Escalate warning classification; update guides removing macros.
3. Major (N+2): Remove implementations, leaving assertion errors if somehow still imported (or ship shim addon). Export stubs that throw instructive error messages during beta period.

### Tooling & Codemods

Codemod goals:
1. Remove imports from `@ember/object/computed` / `@ember/object/compat`.
2. Transform macro property definitions inside `extend({ ... })` hashes and class fields into getters.
3. Insert helper util for multi-prop sort if needed (deduplicate by file).
4. Add TODO for unrecognized dynamic macro usage (e.g., macros stored in variables before use).
5. Ensure no shadowed names when creating backing slots (e.g., `_tempName`).

Lint rules:
- disallow new imports from deprecated modules (error level)
- suggest getter patterns for boolean macro-like expressions
- detect lingering classic dependency key strings passed to `computed()` and flag.

Performance Note: Autotracking's fine-grained dependency tracking plus recomputation on demand negates most previous caching advantages. For hot getters performing heavy work, authors may opt-in to manual caching as shown earlier. When using `TrackedArray` and other tracked collections, mutations are granular (e.g. a push only invalidates dependents once) reducing the need for custom caches.

### Edge Cases & Advanced Patterns

Chained macros: `or('isAdmin', and('isOwner','isActive'))` flattens to boolean expression order preserved with parentheses.

Macros with dynamic keys (rare): `let key = someLogic(); computed.alias(key)` – codemod cannot resolve; replace manually with getter referencing dynamic access `this[someLogic()]` or refactor design.

Macros used in mixins / reopen: Treat like normal properties; conversion to ES classes may be prerequisite (codemod can emit TODO when encountering `Mixin.create({ ... })`).

Setter CPs: If a classic CP supplied a setter distinct from getter semantics, replicate using class getter/setter pair. Example:
```js
fullName: computed('first','last', {
  get() { return `${this.first} ${this.last}`; },
  set(_k, value) { [this.first, this.last] = value.split(' '); return value; }
})
```
->
```js
get fullName() { return `${this.first} ${this.last}`; }
set fullName(value) { [this.first, this.last] = value.split(' '); }
```

CP depending on array length via `.length`: Just access `.length` in getter; autotracking captures it.

Observers relying on CP change events: Removal may simplify away some observer triggers. Guidance: Replace observers with getters + consuming reactive usage or explicit method calls. (Broader observer deprecation handled elsewhere.)

## How We Teach This (Draft)

- Guides: Remove chapters covering CP macros; consolidate derived state docs under a single "Derived State with Getters" section.
- API Docs: Mark each macro as deprecated with guidance snippet linking to deprecation guide.
- Tutorials: Show array derivations using JS array utilities inside getters.
- Lint: New rules discouraging remaining macro imports; autofix suggestions.
- Codemods: Provide automated transforms for common macros (alias/readOnly/reads -> getter; and/or/not/bool -> boolean expressions; collection transforms -> getters with array methods; numeric comparisons -> getters with operators; set operations -> JS Set logic; sort -> [...arr].toSorted / slice+sort; sum/max/min -> reduce/Math.*).

## Drawbacks (Initial)

- Migration effort for large codebases heavily using macros.
- Possible minor performance regressions if naive getters recompute expensive work; mitigated by manual caching patterns (tracked storage, memoization) when necessary.
- Loss of some declarative brevity; traded for standard JS clarity.
- Addon ecosystem needing coordinated releases and codemod runs.

## Alternatives

- Keep a small curated subset (e.g. `alias`, `or`, `and`) – rejected: keeps dual paradigms and complexity surface.
- Deprecate only dependency-key style but keep macros as sugar over getters – rejected: macros still leak dependency key mental model and internal complexity vs value.
- Ship a separate optional addon with macros – possible future community effort if demand persists.

## Unresolved Questions (Initial)

- Final deprecation timeline & staging (minor release introduction vs major removal).
- Whether to provide an official helper library for common boolean/collection helpers (or rely purely on plain JS & community packages).
- Edge codemod coverage for complex chained macro usage.

