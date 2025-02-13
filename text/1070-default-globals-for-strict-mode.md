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

This is an ergonomics-focused RFC. The proposed changes today can be polyfilled via `globalThis['...']` accesses.

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
- must not require lifetime management (e.g.: `setTimeout`)
- must not be a single-word lower-case API, because of potential collision with future [new HTML elements](https://developer.mozilla.org/en-US/docs/Web/HTML/Element)
- if the API is a function, the return value should not be a promise
- must be one one of these lists:
  - https://tc39.es/ecma262/#sec-global-object
  - https://tc39.es/ecma262/#sec-function-properties-of-the-global-object
  - https://html.spec.whatwg.org/multipage/nav-history-apis.html#window
  - https://html.spec.whatwg.org/multipage/indices.html#all-interfaces    
  - https://html.spec.whatwg.org/multipage/webappapis.html


> [!IMPORTANT]  
> Because all function invocations are reactive by default, every function called from these APIs will be re-called when arguments change. 


Given the above criteria, the following should be added to default-available strict-mode scope:

### namespaces / objects

TC39:

- [`globalThis`](https://tc39.es/ecma262/#sec-globalthis) - Already available. 

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=MYewdgzgLgBAJgQygmBeGBvGBzATgU3ygEsxsAuGAcgAt8AbekGKOggQipgF8BuAKH4AeKPgC2AB3pJ8APn4xMGbEwBGCegBUaxCADoAUgGUA8gDk90XKWzEAZgE94SBN27CA9KMnTRsoA&format=gjs)
    ```gjs
    <template>
        {{JSON.stringify @data}}
    </template>
    ```

    </details>

- [`Atomics`](https://tc39.es/ecma262/#sec-atomics)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=MYewdgzgLgBARgVwGZIKYCcYF4ZlQdxgEF10BDATwCFk10AKARgDYBKAbgChRJYEBLMFAAc2XARgBVQSJLkK9RCgwdOAocIDaABgC6YgOxdOAehMwDMetsa3WMABoB5AEowATFe03t9nAFYvRhtWAB44dAA%2BTiIoEABbfmAIADoADxAGdREAGhhtPPdWTk5QqFR4gAcAGzJy6JgYAG8mgHNqkDgyaoAVAAt%2BVNiEpNSOsgATGGzhAF9Z0pNyqtr6oA&format=gjs)
    ```gjs
    const buffer = new ArrayBuffer(16);
    const uint8 = new Uint8Array(buffer);
    uint8[0] = 7;

    // 7 (0111) XOR 2 (0010) = 5 (0101)<br>
    Atomics.xor(uint8, 0, 2)

    <template>
      {{Atomics.load uint8}} === 5
    </template>
    ```

    </details>


- [`JSON`](https://tc39.es/ecma262/#sec-json)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=MYewdgzgLgBAJgQygmBeGBvGBzATgU3ygEsxsAuGAcgAt8AbekGKOggQipgF8BuAKH4AeKPgC2AB3pJ8APn4xMGbEwBGCegBUaxCADoAUgGUA8gDk90XKWzEAZgE94SBN27CA9KMnTRsoA&format=gjs)
    ```gjs
    <template>
        {{JSON.stringify @data}}
    </template>
    ```

    </details>

- [`Math`](https://tc39.es/ecma262/#sec-math)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=FAHgLgpgtgDgNgQ0gPmAAjQb0wczgewCME4AVACwEsBnAOgFklzaoEAPNABjQCY0AWAL6DQAekixEKIA&format=gjs)
    ```gjs
    <template>
      {{Math.max 0 2 4}}
    </template>
    ```

    </details>

- [`Reflect`](https://tc39.es/ecma262/#sec-reflect)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=MYewdgzgLgBAJgQygmBeGBvGBzATgU3ygEsxsAuGAcgAt8AbekGKOgqmAXwG4AoXgDxR8AWwAO9JPgB8vGJgzYmAIwT0AKjWIQAdACV8AM3r5gUHdiLwkKKnkIkyVTp0EB6YeMnDpQA&format=gjs)
    ```gjs
    const data = { greeting: 'hello there' };

    <template>
      {{Reflect.get data 'greeting'}}
    </template>
    ```

    </details>

WHATWG:

- [`localStorage`](https://html.spec.whatwg.org/multipage/webstorage.html#the-localstorage-attribute)

    <details><summary>Example</summary>

    Example[^glimmer-call-bug]
    ```gjs
    <template>
      Current Theme: {{localStorage.getItem 'theme'}}

      <button {{on 'click' @toggleTheme}}>
        Toggle
      </button>
    </template>
    ```

    </details>

- [`sessionStorage`](https://html.spec.whatwg.org/multipage/webstorage.html#the-sessionstorage-attribute)

    <details><summary>Example</summary>

    Example[^glimmer-call-bug]
    ```gjs
    <template>
      Current Theme: {{sessionStorage.getItem 'theme'}}

      <button {{on 'click' @toggleTheme}}>
        Toggle
      </button>
    </template>
    ```

    </details>


### functions / utilities

TC39:

- [`isNaN`](https://tc39.es/ecma262/#sec-isnan-number)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=FAHgLgpgtgDgNgQ0gPmAAjQb0wYgJYBmaAFAOZwD2ARgnACoAWeAzgHQsByCHaADAJQBfQegxoWaLh1HYIcZhGGiMEgrQUBPGZgD0hJSB2RYiFEA&format=gjs)
    ```gjs
    <template>
      {{#if (isNaN 0)}}
        is NaN
      {{else}}
        is falsey
      {{/if}}
    </template>
    ```

    </details>

- [`isFinite`](https://tc39.es/ecma262/#sec-isfinite-number)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=FAHgLgpgtgDgNgQ0gPmAAjQb0wYgJYBmaAFAOZwD2ARgnACoAWeAzgHQsBieAdnpGuWq1GLVgEluBHnwCeASgC%2BC9BjQs0XXpBXYIcZhCUqM6grQMydmAPSEjIa5FiIUQA&format=gjs)
    ```gjs
    <template>
      {{#if (isFinite Infinity)}}
        is Finite
      {{else}}
        is falsey
      {{/if}}
    </template>
    ```

    </details>

- [`parseInt`](https://tc39.es/ecma262/#sec-parseint-string-radix)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=FAHgLgpgtgDgNgQ0gPmAAjQb0wczgewCME4AVACwEsBnAOhgQCdqIBJAOzDQCZaBmAL4DQAekixEKIA&format=gjs)
    ```gjs
    <template>
      {{parseInt 2.3}} 
    </template>
    ```

    </details>

- [`parseFloat`](https://tc39.es/ecma262/#sec-parsefloat-string)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=FAHgLgpgtgDgNgQ0gPmAAjQb0wczgewCME4AVACwEsBnAOhgQCdqIAxApNAJloGYBffqAD0kWIhRA&format=gjs)
    ```gjs
    <template>
      {{parseFloat 2.3}} 
    </template>
    ```

    </details>

- [`decodeURI`](https://tc39.es/ecma262/#sec-decodeuri-encodeduri)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=FAHgLgpgtgDgNgQ0gPmAAjQb0wczgewCME4AVACwEsBnAOgBMIBjfRgVQCUBJNAIgFIArACEAHkIAivAL7S0oAPSRYiFEA&format=gjs)
    ```gjs
    <template>
      {{decodeURI "%5Bx%5D"}} 
    </template>
    ```

    </details>

- [`decodeURIComponent`](https://tc39.es/ecma262/#sec-decodeuricomponent-encodeduricomponent)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=FAHgLgpgtgDgNgQ0gPmAAjQb0wczgewCME4AVACwEsBnAOgBMIBjfRgVQCUBJAYX1nwA7CILBoARAFIArACEAHjIAi4gL6q0oAPSRYiFEA&format=gjs)
    ```gjs
    <template>
      {{decodeURIComponent "%5Bx%5D"}} 
    </template>
    ```

    </details>

- [`encodeURI`](https://tc39.es/ecma262/#sec-encodeuri-uri)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=FAHgLgpgtgDgNgQ0gPmAAjQb0wczgewCME4AVACwEsBnAOggDsBjfAEwgFUAlASTQCIA2gA8AuvwC%2BEtKAD0kWIhRA&format=gjs)
    ```gjs
    <template>
      {{encodeURI "[x]"}} 
    </template>
    ```

    </details>

- [`encodeURIComponent`](https://tc39.es/ecma262/#sec-encodeuricomponent-uricomponent)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=FAHgLgpgtgDgNgQ0gPmAAjQb0wczgewCME4AVACwEsBnAOggDsBjfAEwgFUAlASQGF8sfA0Zg0AIgDaADwC64gL4K0oAPSRYiFEA&format=gjs)
    ```gjs
    <template>
      {{encodeURIComponent "[x]"}} 
    </template>
    ```

    </details>

WHATWG:

- [`atob`](https://html.spec.whatwg.org/multipage/webappapis.html#dom-btoa)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=DwFwpgtgDgNghuAfAKAATtQb0wcxgewCM4YAVACwEsBnAOgSNQCI4BxANWsNYA4BeJgF9ByYAHpw0eEmRA&format=gjs)

    ```gjs
    <template>
        {{atob "aGVsbG8="}}
    </template>
    ```

    </details>

- [`btoa`](https://html.spec.whatwg.org/multipage/webappapis.html#dom-atob)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=DwFwpgtgDgNghuAfAKAATtQb0wcxgewCM4YAVACwEsBnAOkJHzlQCJywYCWBfb5YAPTho8JMiA&format=gjs)

    ```gjs
    <template>
        {{btoa "hello"}}
    </template>
    ```

    </details>

- [`postMessage`](https://html.spec.whatwg.org/multipage/web-messaging.html#dom-window-postmessage)

    <details><summary>Example</summary>

    ```gjs
    <template>
        <button {{on "click" (postMessage "Hello")>
            Greet
        </button>
    </template>
    ```

    </details>

- [`structuredClone`](https://html.spec.whatwg.org/multipage/structured-data.html#dom-structuredclone)

    <details><summary>Example</summary>

    ```gjs
    <template>
        <SomeComponent @data={{structuredClone @inputData}} />
    </template>
    ```


    </details>


### new-less constructors (still functions / utilities)

TC39:

- [`Array`](https://tc39.es/ecma262/#sec-constructor-properties-of-the-global-object-array)

    <details><summary>Example</summary>

    See note[^reactive-array-note] about `(array)`, `@ember/helper`, and [RFC#1000](https://github.com/emberjs/rfcs/pull/1000)[^reactive-array-note]

    [Link](https://limber.glimdown.com/edit?c=DwFwpgtgDgNghuAfAKAATtQb0wYjHAYwAtUAKAcxgHsAjOGAFSIEsBnAOgEEAnbuAT1QBGVACZUAZgCUqOK1QAfZuAgKAvmrQZt2ZZA1b02APT5iB4MZWwEYFEA&format=gjs)
    
    ```gjs
    <template>
        {{#each (Array 1 2 3) as |item|}}
            {{item}}
        {{/each}}
    </template>
    ```

    [^reactive-array-note]: This is the same behavior as `(array)` in loose mode, and `import { array } from '@ember/helper';`, however, while the creation of the array is reactive (e.g.: if we had said `(Array @foo @bar)`, changes to `@foo` and `@bar` would cause the creation of a new array instance), the proposed _built-in_ `(array)` _keyword_ behavior _may_ have reactive items, as proposed by [RFC#1068](https://github.com/emberjs/rfcs/pull/1068) 

    </details>



- [`BigInt`](https://tc39.es/ecma262/#sec-constructor-properties-of-the-global-object-bigint)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=DwFwpgtgDgNghuAfAKAATtQb0wcxgewCM4YAVACwEsBnAOgCFKcBJAOxFQCIAGADwEYAZsJGiRnAL4TkwAPTho8JMiA&format=gjs)
    ```gjs
    <template>
        {{BigInt "0x1fffffffffffff"}}
    </template>
    ```

    </details>

- [`Boolean`](https://tc39.es/ecma262/#sec-constructor-properties-of-the-global-object-boolean)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=DwFwpgtgDgNghuAfAKAATtQb0wYgJYBmqAFAOYwD2ARnDACoAWeAzgHQBCFFMYcAdqgBEggJQBfMWgzoQAJwCuYKemxgYzMBOUYCtDduwB6QluCHw0eEmRA&format=gjs)
    ```gjs
    <template>
        {{#if (Boolean "")}}
          true
        {{else}}
          false
        {{/if}}
    </template>
    ```

    </details>

- [`Date`](https://tc39.es/ecma262/#sec-constructor-properties-of-the-global-object-date)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=DwFwpgtgDgNghuAfAKAATtQb0wcxgewCM4YAVACwEsBnAOgBEExUBGAdgGY2AmAFg4AMAgKwBOPgF8JyYAHpw0eEmRA&format=gjs)
    ```gjs
    <template>
        {{Date 1737243005924}}
    </template>
    ```

    </details>

- [`Number`](https://tc39.es/ecma262/#sec-constructor-properties-of-the-global-object-number)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=DwFwpgtgDgNghuAfAKAATtQb0wcxgewCM4YAVACwEsBnAOgDkBXCQsAJ1QCIAmbzgX37JgAenDR4SZEA&format=gjs)
    ```gjs
    <template>
        {{Number "22"}}
    </template>
    ```

    </details>

- [`Object`](https://tc39.es/ecma262/#sec-constructor-properties-of-the-global-object-object)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=DwFwpgtgDgNghuAfAKAATtQb0wYjHAYwAsBaASwDtUAKAcxgHsAjOGAFSLIGcA6AeSYArMARCo4AXgCMqJhIBMASnFdUAHwDWYAJ6oAbqwCuYNQF9TaDOmxbt5gFxZMBmMfOXrmAPT5i5Cu7AXuDQ8EjIQA&format=gjs)
    ```gjs
    <template>
        {{#each-in (Object a=1 b=2) as |key value|}}
          {{key}}: {{value}}
        {{/each-in}}
    </template>
    ```

    </details>

- [`String`](https://tc39.es/ecma262/#sec-constructor-properties-of-the-global-object-string)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=DwFwpgtgDgNghuAfAKAATtQb0wcxgewCM4YAVACwEsBnAOgGUQAnSgOx1QCZOBfH5YAHpw0eEmRA&format=gjs)
    ```gjs
    <template>
        {{String 22}}
    </template>
    ```

    </details>

### Values

TC39:

- [`Infinity`](https://tc39.es/ecma262/#sec-value-properties-of-the-global-object-infinity)

    <details><summary>Example</summary>

    ```gjs
    <template>
        {{Infinity}}
    </template>
    ```

    </details>

- [`NaN`](https://tc39.es/ecma262/#sec-value-properties-of-the-global-object-nan)

    <details><summary>Example</summary>

    ```gjs
    <template>
        {{NaN}}
    </template>
    ```

    </details>

WHATWG:

- [`isSecureContext`](https://html.spec.whatwg.org/multipage/webappapis.html#dom-issecurecontext)

    <details><summary>Example</summary>

    ```gjs
    <template>
        {{#if isSecureContext}}
            is secure
        {{/if}}
    </template>
    ```

    </details>



### Potentially matching criteria, but not included

Existing keywords don't need to be included in the global scope allow-list

- [`undefined`](https://tc39.es/ecma262/#sec-undefined)

These do not exist in all supported environments:

- [`Encode`](https://tc39.es/ecma262/#sec-encode)
- [`Decode`](https://tc39.es/ecma262/#sec-decode)
- [`ParseHexOctet`](https://tc39.es/ecma262/#sec-parsehexoctet)

These are not common and / or may be actively dangerous to make easier to use:

- `eval`
- `PerformEval`
- `Function`

Uncommon entries from the "Constructor Properties" list: https://tc39.es/ecma262/#sec-constructor-properties-of-the-global-object
- `${...}Error`, e.g.: `AggregateError`
- `${...}Int${...}Array`, e.g.: `BigUint64Array`,
- anything that requires `new`, e.g.:  `DataView`, `Map`

APIs from WHATWG that are highly likely to collide with user-land code or are already ambiguous (and thus would be confusing to use):

- `stop()`, `close()`, `status()`, `focus()`, `blur()`, `open()`, `parent`, `confirm()`, `self`, etc


## How we teach this

Developers should primarily reference exising documentation on the web for the above-mentioned APIs, such as on MDN.

If we don't already, we should have an extensive guide on Polish Syntax, potentially similar to [https://cheatsheet.glimmer.nullvoxpopuli.com/docs/templates](https://cheatsheet.glimmer.nullvoxpopuli.com/docs/templates)

## Drawbacks

Takes a small amount of work to implement.


## Alternatives

- Do nothing, but this is worse, as folks intuitively expect these a lot of the above-mentioned APIs to "just work", without needing weird scope-tricks to convince our Scope-tracking tools in the build tools that certain APIs are in scope..

- Add some single-word globals, despite the risk of HTML element being implemented with the same word / casing


- [`location`](https://html.spec.whatwg.org/multipage/nav-history-apis.html#dom-location)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=FAHgLgpgtgDgNgQ0gPmAAjQb0wczgewCME4AVACwEsBnAOgIGMlL8A7W8gJwgDM0BffqAD0kWIhRA&format=gjs)
    ```gjs
    <template>
      {{location.href }}
    </template>
    ```

    </details>

- [`history`](https://html.spec.whatwg.org/multipage/nav-history-apis.html#dom-history)

    <details><summary>Example</summary>

    Example[^glimmer-call-bug]
    ```gjs
    import { on } from '@ember/modifier';

    <template>
      <button {{on 'click' history.back}}>Back</button>
    </template>
    ```
    [^glimmer-call-bug]: demo/example omitted because the glimmer-vm, at the time of the writing of this RFC has not fixed a bug that where this-binding is lost on method calls.

    </details>

- [`navigator`](https://html.spec.whatwg.org/multipage/system-state.html#dom-navigator)

    <details><summary>Example</summary>

    [Link](https://limber.glimdown.com/edit?c=FAHgLgpgtgDgNgQ0gPmAAjQb0wczgewCME4AVACwEsBnAOgDsEA3SnJfAJ1oFdqIOAgjgj0wAXzGgA9JFiIUQA&format=gjs)
    ```gjs
    <template>
      {{navigator.userAgent}}

      <button {{on 'click' (fn navigator.clipboard.write @blob)}}>
        Copy to clipboard
      </button>
    </template>
    ```

    </details>

- [`window`](https://html.spec.whatwg.org/multipage/nav-history-apis.html#dom-window)

    Most APIs are also available on `window`

- [`document`](https://html.spec.whatwg.org/multipage/nav-history-apis.html#dom-document-2)

    <details><summary>Example</summary>

    Example[^glimmer-call-bug]
    ```gjs
    <template>
      <div id='foo'></div>
      
      {{#in-element (document.getElementById 'foo')}}
          rendered elsewhere
      {{/in-element}}
    </template>
    ```

    </details>




## Unresolved questions

none
