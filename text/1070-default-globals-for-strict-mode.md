---
stage: accepted
start-date: 2025-01-18T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - framework
  - learning
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1070 
project-link:
suite: 
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
suite: Leave as is
-->

# Default globals for strict mode 

## Summary

This RFC aims to introduce platform-native globals as allowed defaults in strict-mode components, allowing for more intuitive usage, and less "know how the compiler works" 

## Motivation

Early on there was a bug in the build tools of strict-mode components that allowed _any_ global to be used components. This was dangerous, as strict-mode allows use of all in-scoped variables to be used in all valid syntax positions in the template, and while this is what folks would expect for languages with a scope-system, it means that if someone defined `window.div`, all `<div>`s in components would use that implementation instead.  This was fixed, but during that time, we realized that it's _very_ convenient to use platform-native globals as utilities, such as `Array`, `Boolean`, `String`, `Infinity`, `JSON`, and many more.

While we don't want to support _everything_ on `globalThis`, we can aim support a good list of utilities fitting some criteria. Committing to this list means that we promise to never create a keyword with the same name + casing as the platform-native API, likewise, having an allow-list of which platform-native APIs to support guides our decisions around what keywords to implement in templates, as the platform-native globals would take precedence over / be used instead of any same-named keywords. 

Without this RFC, all platform-native globals must be accessed via globalThis:

```gjs
<template>
  {{globalThis.JSON.stringify @data null 3}}
</template>
```
or
```gjs
const { JSON } = globalThis;

<template>
  {{JSON.stringify @data null 3}}
</template>
```

After this RFC is implemented, the following would work:
```gjs
<template>
  {{JSON.stringify @data null 3}}
</template>
```

## Detailed design

Allowing defaults means: when using `JSON` (for example) in a component, the compiled-to-plain-JS output results in the reference to JSON being added to the "scope bag", for example: :

```js
// Post-RFC 931
import { template } from '@ember/template-compiler'; 

const data = {};

export default template('{{JSON.stringify data}}', { scope: () => ({ JSON, data }) });
```

<details><summary>pre-RFC#931</summary>

```js
// Pre-RFC  931
import { precompileTemplate } from "@ember/template-compilation";
import { setComponentTemplate } from "@ember/component";
import templateOnly from "@ember/component/template-only";

const data = {};

export default setComponentTemplate(
    precompileTemplate('{{JSON.stringify data}}', { 
        strictMode: true, 
        scope: () => ({ JSON, data }) }
    ), templateOnly()
);
```

</details>

Criteria for a platform-native global to be accepted as default:

Any of
- begins with an uppercase letter 
- guaranteed to never be added to glimmer as a keyword (e.g.: `globalThis`)

And
- must not need `new` to invoke
- must be one one of these lists:
  - https://tc39.es/ecma262/#sec-global-object
  - https://tc39.es/ecma262/#sec-function-properties-of-the-global-object
  - https://html.spec.whatwg.org/multipage/nav-history-apis.html#window


Given the above criteria, the following should be added to default-available strict-mode scope:

### namespaces

- `globalThis`
- `JSON`
- `Math`
- `Atomics`
- `Reflect`

### functions / utilities

- `isNaN`
- `isFinite`
- `parseInt`
- `parseFloat`
- `decodeURI`
- `decodeURIComponent`
- `encodeURI`
- `encodeURIComponent`

### new-less constructors (still functions / utilities)

- `Number`
- `Object`
- `Array`
- `String`
- `BigInt`
- `Date`

### Values

- `Infinity`





> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here. 

> Please keep in mind any implications within the Ember ecosystem, such as:
> - Lint rules (ember-template-lint, eslint-plugin-ember) that should be added, modified or removed
> - Features that are replaced or made obsolete by this feature and should eventually be deprecated
> - Ember Inspector and debuggability
> - Server-side Rendering
> - Ember Engines
> - The Addon Ecosystem
> - IDE Support
> - Blueprints that should be added or modified

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

> Keep in mind the variety of learning materials: API docs, guides, blog posts, tutorials, etc.

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
