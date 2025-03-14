---
stage: ready-for-release
start-date: 2025-01-10T00:00:00.000Z
release-date:
release-versions:
teams:
  - cli
  - data
  - framework
  - learning
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1065'
project-link:
---

<!---
Directions for above:

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
-->

# Deprecate and Remove ember-fetch 

## Summary

This RFC proposes removing `ember-fetch` from the blueprint for new projects, and recommends alternative, more native, ways of having "managed fetch".

## Motivation

The package, `ember-fetch`, does a fair bit of deceptive and incorrect behavior that is incompatible with modern JavaScript tooling, such as _being_ `ember-fetch`, yet only importing from `fetch`.

## Transition Path

- Remove ember-fetch from all blueprints
- Deprecate the npm package and archive the github repo.
- Migrate to an alternative for "managed fetch" 

### What does `ember-fetch` do?

_Primarily_, it wraps the native `fetch` in `waitForPromise` from `@ember/test-waiters` (aka "settled state integration").


Secondarily, but not popularly used, are a series of utilities (e.g.: for checking kinds of errors). These could be copied into projects that use them and modified to fit each project's needs. 

### Using native `fetch`

If you only use `ember-fetch` in route model-hooks, then the settled-state integration isn't used (or rather, you rely on the routing's settled-state integration, and `ember-fetch`'s settled-state integration is extraneous) 


### Direct replacement


To replace `ember-fetch`'s core-functionality using the least amount of effort involves adding a new utility in `@ember/test-waiters`, `waitForFetch`:

```ts
// in @ember/test-waiters
import { waitForPromise } from './wait-for-promise';

export async function waitForFetch<Value>(fetchPromise: Promise<Value>) {
    waitForPromise(fetchPromise);

    let response = await fetchPromise;

    return new Proxy(response, {
        get(target, prop, receiver) {
            let original = Reflect.get(target, prop, receiver);

            if (['json', 'text', 'arrayBuffer', 'blob', 'formData'].includes(prop)) {
                return (...args) => {
                    let parsePromise = original.call(target, ...args);

                    return waitForPromise(parsePromise);
                }
            }

            return original;
        }
    });
}
```


To have a single import mirroring the behavior of `ember-fetch`, _in full_, folks can still provide a single export that does:

```ts
import { waitForFetch } from '@ember/test-waiters';

export function wrappedFetch(...args: Parameters<typeof fetch>) {
    return waitForFetch(fetch(...args));
}
```


And then throughout your project, you could find and replace all imports of `import fetch from 'fetch';` with `import { wrappedFetch } from 'app-name/utils/wrapped-fetch';`



### Using a service

Potentially an easier-to-manage implementation uses a service such as the following:

```ts
import Service from '@ember/service';
import { waitFor } from '@ember/test-waiters';

export default class Fetch extends Service {

    @waitFor
    @action
    requestJson(...args: Parameters<typeof fetch>) {
        return fetch(...args).then(response => response.json());
    }

    @waitFor
    @action
    requestText(...args: Parameters<typeof fetch>) {
        return fetch(...args).then(response => response.text());
    }
}
```

### Using warp-drive / ember-data

Docs available on https://github.com/emberjs/data/

## How We Teach This

- Add a deprecation notice to the ember-fetch README
- Archive the ember-fetch repo
- Remove ember-fetch from the blueprints
- Remove ember-fetch from the guides (only one reference per version)

## Drawbacks

- if we keep ember-fetch is not compatible with modern tooling and modern SSR approaches

## Alternatives

- n/a
- ask folks to wrap and proxy fetch themselves

## Unresolved questions

none
