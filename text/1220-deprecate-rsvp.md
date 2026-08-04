---
stage: accepted
start-date: 2026-08-04T00:00:00.000Z
release-date:
release-versions:
teams: # delete teams that aren't relevant
  - framework
  - learning
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1220
project-link:
---

# Deprecate RSVP

## Summary

Deprecate the `rsvp` module bundled with `ember-source`.

Native `Promise` has been in every browser and node version we support for a long time now, and covers nearly everything RSVP does. 
[`ember-data` already did this](https://github.com/emberjs/rfcs/pull/796) back in 2022.

The [`rsvp` package on npm](https://www.npmjs.com/package/rsvp) itself is unaffected by this deprecation, but we recommend migrating to native `Promise` rather than adding a direct dependency on `rsvp`.

## Motivation

RSVP is Ember's Promises/A+ implementation from before browsers had one, and native `Promise` has since made almost all of it redundant.

Deprecating it:
- slims down our public API surface area to more of _what's needed_
- removes bytes from every app (RSVP is bundled with `ember-source` whether you use it or not)
- removes one of the remaining ties to the runloop (`ember-source` configures RSVP to schedule promise resolution via backburner, which blocks eventually removing the runloop)
- removes "another case to cover" for tooling, types, and teaching. New folks should only ever have to learn native `Promise`

## Transition Path

Most usage is a mechanical find-and-replace:

| RSVP | Native |
| ---- | ------ |
| `RSVP.Promise` / `import { Promise } from 'rsvp'` | `Promise` |
| `RSVP.resolve(x)` | `Promise.resolve(x)` |
| `RSVP.reject(x)` | `Promise.reject(x)` |
| `RSVP.all(array)` | `Promise.all(array)` |
| `RSVP.race(array)` | `Promise.race(array)` |
| `RSVP.allSettled(array)` | `Promise.allSettled(array)` [^settled] |
| `RSVP.defer()` | [`Promise.withResolvers()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/withResolvers) [^defer] |
| `RSVP.hash(obj)` | no native equivalent, see below |
| `RSVP.map(array, fn)` | `Promise.all(array.map(fn))` |
| `RSVP.filter(array, fn)` | `Promise.all` + `Array.prototype.filter` |
| `RSVP.denodeify(fn)` | [`util.promisify`](https://nodejs.org/api/util.html#utilpromisifyoriginal) (node), or wrap in `new Promise` |
| `RSVP.EventTarget` | native [`EventTarget`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget) |
| `RSVP.on('error', fn)` | [`unhandledrejection`](https://developer.mozilla.org/en-US/docs/Web/API/Window/unhandledrejection_event) event |
| `RSVP.rethrow` | not needed, devtools handle async stack traces now |

[^settled]: the result objects differ slightly: RSVP uses `{ state: 'fulfilled' }`, native uses `{ status: 'fulfilled' }`.

[^defer]: returns `{ promise, resolve, reject }`, the same shape as `RSVP.defer()`.

`RSVP.hash` is the only utility without a native equivalent, and it's a one-liner:

```js
async function hash(obj) {
  return Object.fromEntries(
    await Promise.all(
      Object.entries(obj).map(async ([key, promise]) => [key, await promise])
    )
  );
}
```

(or write it inline; two `await`s is often clearer than `hash` anyway)

<details><summary>example codemod-ish diff</summary>

```diff
-import RSVP from 'rsvp';
+
 
 export default class MyService extends Service {
   async loadEverything() {
-    return RSVP.hash({
-      user: this.store.request(findRecord('user', 1)),
-      settings: fetch('/settings').then((r) => r.json()),
-    });
+    let [user, settings] = await Promise.all([
+      this.store.request(findRecord('user', 1)),
+      fetch('/settings').then((r) => r.json()),
+    ]);
+
+    return { user, settings };
   }
 }
```

</details>

### Timing differences

`ember-source` configures RSVP so that promise resolution is flushed by the runloop. Native promises use the browser's microtask queue directly.
In practice these are nearly indistinguishable (backburner has been microtask-based since ember-source@3.x), but:

- test code that relied on `await settled()` "seeing" pending RSVP chains should use [`@ember/test-waiters`](https://github.com/emberjs/ember-test-waiters) for any async that renders
- code that relied on `RSVP.on('error')` for global error reporting should use the `unhandledrejection` event

### Deprecation mechanics

- the `rsvp` module provided by `ember-source` issues a runtime deprecation (via the existing deprecation system, `deprecate` from `@ember/debug`) when any of its exports are used
  - `id: deprecate-rsvp`, `until: 8.0.0`
- the deprecation only covers the copy bundled with `ember-source`. Installing `rsvp` from npm directly would silence it, but migrating to native `Promise` is the recommended path
- `ember-source`'s internals (`ember-testing`, router promise handling) migrate to native promises. This is not observable, except via `instanceof RSVP.Promise` checks, which nobody should be doing
- a lint rule should be added to `eslint-plugin-ember`'s recommended config flagging `rsvp` imports

### Deprecation guide

> `RSVP` is deprecated. Use native `Promise` instead.
>
> Before:
> ```js
> import RSVP from 'rsvp';
>
> await RSVP.all(promises);
> ```
>
> After:
> ```js
> await Promise.all(promises);
> ```
>
> We recommend migrating to native `Promise` rather than adding a dependency on the `rsvp` npm package; everything RSVP provides has a native equivalent or a small inline replacement.

## How We Teach This

The guides and blueprints already use native promises and `async`/`await` everywhere.

Remaining work:
- add the deprecation guide entry to https://deprecations.emberjs.com
- mark the `rsvp` module as deprecated in the API docs

Overall, this reduces what we have to teach, since there is only one kind of promise left.

## Drawbacks

As with any deprecation, we introduce an upgrade cliff for addons that are updated infrequently, and consequently their consuming apps.

Unlike most deprecations, though, the replacement (`Promise`) works in every supported Ember version, so addons can migrate today without `@embroider/macros` and without narrowing their supported version range.

The bigger cost is timing-sensitive test suites that implicitly depend on RSVP's runloop scheduling. Those need `@ember/test-waiters`, which they should be using regardless of this RFC.

## Alternatives

do nothing, the cost of bundling RSVP is:
- bytes in every app, used or not
- a permanent tie between promise resolution and the runloop
- mental gymnastics for teaching ("use native promises, except this module the framework ships is a different kind of promise")
- "another case to cover" for tooling and types

deprecate only the runloop integration, keep re-exporting `rsvp`
- solves the runloop problem, keeps all the other costs
- at that point `ember-source`'s copy of RSVP has no behavioral difference from the npm package, so re-exporting it serves no purpose

add a lint against `rsvp` imports without deprecating
- all the downsides of "do nothing" may still be present

## Unresolved questions

n/a
