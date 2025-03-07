---
stage: accepted
start-date: 2025-03-06T00:00:00.000Z # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - framework
  - learning
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1079 # Fill this in with the URL for the Proposal RFC PR
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

<!-- Replace "RFC title" with the title of your RFC -->

# Propose a simpler tracked state utility from `@ember/reactive` 

## Summary

The RFC proposes solving two things:
- import fatigue
- reactive-wrapper fatigue 

by introducing a brand new package (one additional import: `@ember/reactive`) with a named export, `reactive` which allows deep reactivity across all our default reactive data structures in the current blueprint:
- primitive values (only possible via decorator)
- arrays
- objects
- maps
- sets
- weak maps
- weak sets



> [!NOTE]  
> It is also possible we can introduce additional defaults in the future. Such as the [promise, proposed (sort of) in RFC #1060](https://github.com/emberjs/rfcs/pull/1060)

## Motivation

Example[^example-requires-vm-work]:
```gjs
import { reactive } from '@ember/reactive';

const data = reactive({ greeting: 'hi' });
const exclaim = () => data.greeting += '!'; 

export const Demo = <template>
    {{data.greeting}} === 'hi'
    <button {{on "click" (fn exclaim data)}}>Exclaim</button>
</template>;
```

<details><summary>class-based example</summary>

Same footnote applies as above[^example-requires-vm-work]

```gjs
import Component from '@glimmer/component';
import { reactive } from '@ember/reactive';

export class Demo extends Component {
    // makes the *reference* reactive
    @reactive count = 0;

    // makes the right-hand side of the = reactive
    data = reactive({ greeting: 'hello' });

    // both left and right-hand sides are reactive
    @reactive data2 = { greeting: 'hello' };

    exclaim() {
        this.count++;
        this.data.greeting += "!";
        this.data2.greeting += "!";
    }

    <template>
        {{this.count}}
        {{this.data.greeting}}
        {{this.data2.greeting}}

        <button {{on "click" this.exclaim}}>Exclaim</button>
    </template>
}
```

</details>

[^example-requires-vm-work]: These examples assume that these RFCs are implemented: [#989: `fn` as a keyword](https://github.com/emberjs/rfcs/pull/998), [#997: `on` as a keyword](https://github.com/emberjs/rfcs/pull/997), [#1070: Default globals for strict mode](https://github.com/emberjs/rfcs/pull/1070), and that the long-standing bug in the VM where plain methods don't work in class-based components is fixed.

Feedback on prior RFCs, [Tracking utilities for promises](https://github.com/emberjs/rfcs/pull/1060), [The Cell](https://github.com/emberjs/rfcs/pull/1071), and [tracked-built-ins, built-in](https://github.com/emberjs/rfcs/pull/1068), revealed that folks have found that building reactive data structures, by default, is cumbersome. While there utilities would not be removed in anyway, we want to provide a way for folks to make things easily reactive, and _still_ provide a way to get fine-grained reactivity via the existing tools that folks are used to today.

The main tradeoff with deep reactivity vs explicit / shallow is the impact to memory. This RFC describes those tradeoffs, and provides a way for folks to determine when they may want to consider 

This RFC could, but is not required to, replace: 
- [RFC#1068](https://github.com/emberjs/rfcs/pull/1068): tracked-built-ins, built-in

## Detailed design

Goals for `reactive`:
- decorator and non-decorator usage (compatible with typescript)
- handles both primitive values and the _common collections_[^the-common-collections]
- deeply reactive[^deep-reactivity] by default (and lazily deeply reactive) 
- Since deep reactivity has a memory cost, we also want a shallow version which doesn't infinitely proxy to any depth 

[^the-common-collections]: The comment collection data structures are: `Array`, `Set`, `WeakSet`, `Object`, `Map`, `WeakMap`
[^deep-reactivity]: this is to make working with nested data easier, and reduce bugs encountered when updating _parts_ of state. The main known cost is to memory, as we have to make use of proxies around every value so we can continue to be deeply reactive, and lazily reactive.

### Proposed API / available usage

[Example Types at tsplay.dev](https://www.typescriptlang.org/play/?experimentalDecorators=true&target=99&jsx=0#code/JYWwDg9gTgLgBAbzgUwB5mQYxgFQJ4YDyAZnAL5zFQQhwDkaG2AtDAcnQNwBQ3A9HzgALGDDABnAFwCA5sBhCArgCMAdJhp9kIZcigArcXyjFMRsIoA2lvgEYADAHZbfYsEvJx3YADsYe4gBDTGQ4ACVkYJhgADdkAB4ANUDLRWQAPkRuODhMRSgoZD9JOGTU5B4cwsCAEwAKAEoSsrSeMl5ffxNg0IBhZGsklLTMtH8fGvFwyOxYhJaMrJyBOFU17nbuNgw4AAUoUHk5uABeOB9FHT04AB84cRgDnxlb86tLV8UJ5DcfZBrXsoIBAPIEfDx+IIwJZAr44IpxIEZMhuMQvrMID44NVZnEhuV0nVfBYYM1hsgmqVyShUONJnsDiAjnE4AB+OD9QYLTJk8o8FYPJGhFwAJjgNSw0ECMGgqPR0Ux2Jm0TidRggSgyNJcAgyn0WBgABo4ABrZB4EoPJ4vO7iPA6EHGiXiTAHMAyqCskpfE0+CAAdx8lLBeH5gnETDgghwvQAzABOOAAEUlUGlsrRPgxWJxKvm5MJ2Tg6s1yG1vRh4nEAEFMCEq9AUxo0x6cBqtfEfX7A8bucaixo-GMShXAlXa-XxI3U+moL1Mf5adxKaPx3XPFOoE2pR6IuIrDBOz5fQGfL2CxCVqBodoiuqFVi1exxRBPD46PAQNKusa0FFLHgDRylmD5KlEcz4iMdRrKo7ZSPCx7dj4ADaAC6lJdqeSxKjA+RYvYcBjghJ6BoRUwLG0vCQgING0bRkJwDgngwOIACE1F0ZxfDeEO3QhHAADKNDIIQerYcQwIlBcVxQJUcDKBqUnIHEskbLwKy6vq2DcIwBr4EQxB1Lmcx1EgEkQCUYpkA0DSqDKACy0qYEIolabg7DxGZklvDJ5CEg0YaEQUgR4Dp6B6ewJBGcqJnIXQ5l0OhtkygAogAjooKT6SJxDxFavgyGh-mBV+YBhUw7kGdF4Gqn8-pwI5YB5Y8BXGvlzz+TZdkQOlmWWNlJDxI1zXWm1LUdcVDH+pEJpwKV5URVVxm1cg9UAOozcNQkgCJerGkCIKRD4nXJT1GVZZFuUbYEJpbcJrn7cCoLHZNAplgt2ADYZy3IHUdWCWWI0FSd3W9RdBnxAJgPtTI6SvYI003fc726Z9l3Vbiv3-ddJpQ4e227foIOped-WXfEON45D916nDjSXoIErNrOXiMNA8CYJWUwAGLAthKxAgoYGY0qxB6EU-FggCP1wDE5JFgAAjLGhfPAZz2HJSsxSygSnIglDeVZmsy8oetxQlqHG9roSYHr-3DTDY3WpNORazVoQAmc2ObYETUEw98lPUdLtwG7IuhF7q0A4eMMh2HeaUHbUeU4D-u0-TvDLIIMty+UOo+ABxqFGLhRZqEwBTIK0SYEWLxnD9dQEUR0m6FAAVFkIesN15FlwCK1lycAXfW3U5vAol7c5Pow-u39UcO+NMhO8DjRATks31yP3s3XdO0Bwdz2dXJHyb7P-1U7Hq9ybQp+Y3P60zVTadE1fmdRoIjCWMAmDyMLCfF+LMuhEJh-2OLnNIisZZYlvnmRuZEfKt0nqHGWEAZ53x7pZAekDrZwDAGg2BY8IATytu7OAaV8EmXtr7IGzxl4dVXtg0hUAKErQfjvahz9HqHTBEfRhIspgwMoVHC+i9eGuxlmrUBrC4Ap3xjTF+NlKLcA8PAUgkd6q8wgBnD6lUcp1GIOoCAqtTpgzJhDFuehJqo10VFAxgQTGky+p5cyUlLitz8hnaxX19GqGUA4vqTiYZFU8eFNGVUDGYH8eDHKQ1qGO3uKIqxoSbGGQMTUKJZiYk413oTLhh8kkVW8QYikoNHHkxEc7Api09EGOIBkpxsjqZ71ppNHRRTVAyHqeTTklh4gWKgHTduXj0YGKEF0iGCAXEIOuGQKpYSamqGAOMmJQTUJzJST4-QyzBoL1GgkypITCkjNUCabZV0fZ+3kXk4O6z2mWDOZDaGiTDnVNsaoEADzGnP0GRCYZ4TVBBlKQE7pAxen9J+W045EAHmTO8v0jxQzkntLAA81ZtzjlpQebs1q+zgbov+W3IF0TBrZI4VcwO3CXovPmW88QDyKl4upRsgxMBPmP1TvIiF3AgA)

<details><summary>example type tests</summary>

```ts
import { expectTypeOf } from 'expect-type';

// https://github.com/emberjs/rfcs/pull/1071/files
interface Reactive<Value> {
  current: Value;
  read(): Value;
}

interface Cell<Value> extends Reactive<Value> {
  // ...
}

type Primitive = number | string | null | undefined | boolean;

// plain usage
function reactive<Value>(input: Value): Value extends Primitive ? Cell<Value> : Value;
// stage 1/2 decorator
function reactive(target: object, key: string | symbol, descriptor?: unknown): any;
// spec / TC39 Decorator
function reactive<Value>(
  target: ClassAccessorDecoratorTarget<unknown, Value>, 
  context: ClassAccessorDecoratorContext
): ClassAccessorDecoratorResult<unknown, Value>;

// implementation (type doesn't matter, exactly)
function reactive<Value>(...args: unknown[]): unknown {
  return 0 as unknown as Value;
}


///////////////
// Tests!
///////////////
interface SomeObj {
  foo: number;
  bar: never;
}

// object
expectTypeOf(reactive({ foo: 2 })).toMatchObjectType<{ foo: number }>();
// array
expectTypeOf(reactive(['foo'])).toEqualTypeOf<string[]>();
// map
expectTypeOf(reactive(new Map<string, string>())).toEqualTypeOf<Map<string, string>>();
// weak map
expectTypeOf(reactive(new WeakMap<SomeObj, boolean>())).toEqualTypeOf<WeakMap<SomeObj, boolean>>();
// set
expectTypeOf(reactive(new Set<string>())).toEqualTypeOf<Set<string>>();
// weak set
expectTypeOf(reactive(new WeakSet<SomeObj>())).toEqualTypeOf<WeakSet<SomeObj>>();

// decorators
export class Foo {
  // both reactive reference and reactive value
  @reactive count = 0;
  @reactive a = { foo: 2 };
  @reactive b = ['foo'];
  @reactive c = new Map<string, string>();
  @reactive d = new WeakMap<SomeObj, boolean>();
  @reactive e = new Set<string>();
  @reactive f = new WeakSet<SomeObj>();

  // reactive value only, reference is static
  g = reactive(0 as number);
  h = reactive({ foo: 2});
  i = reactive(['foo']);
  j = reactive(new Map<string, string>())
  k = reactive(new WeakMap<SomeObj, boolean>());
  l = reactive(new Set<string>());
  m = reactive(new WeakSet<SomeObj>());

  // explicit reactive reference and reactive value
  @reactive n = reactive(0 as number);
  @reactive o = reactive({ foo: 2});
  @reactive p = reactive(['foo']);
  @reactive q = reactive(new Map<string, string>())
  @reactive r = reactive(new WeakMap<SomeObj, boolean>());
  @reactive s = reactive(new Set<string>());
  @reactive t = reactive(new WeakSet<SomeObj>());
}

let f = new Foo();

expectTypeOf(f.count).toEqualTypeOf<number>();
expectTypeOf(f.a).toEqualTypeOf<{foo: number }>();
expectTypeOf(f.b).toEqualTypeOf<string[]>();
expectTypeOf(f.c).toEqualTypeOf<Map<string, string>>();
expectTypeOf(f.d).toEqualTypeOf<WeakMap<SomeObj, boolean>>();
expectTypeOf(f.e).toEqualTypeOf<Set<string>>();
expectTypeOf(f.f).toEqualTypeOf<WeakSet<SomeObj>>();

expectTypeOf(f.g).toEqualTypeOf<Cell<number>>();
expectTypeOf(f.h).toEqualTypeOf<{foo: number }>();
expectTypeOf(f.i).toEqualTypeOf<string[]>();
expectTypeOf(f.j).toEqualTypeOf<Map<string, string>>();
expectTypeOf(f.k).toEqualTypeOf<WeakMap<SomeObj, boolean>>();
expectTypeOf(f.l).toEqualTypeOf<Set<string>>();
expectTypeOf(f.m).toEqualTypeOf<WeakSet<SomeObj>>();

expectTypeOf(f.n).toEqualTypeOf<Cell<number>>();
expectTypeOf(f.o).toEqualTypeOf<{foo: number }>();
expectTypeOf(f.p).toEqualTypeOf<string[]>();
expectTypeOf(f.q).toEqualTypeOf<Map<string, string>>();
expectTypeOf(f.r).toEqualTypeOf<WeakMap<SomeObj, boolean>>();
expectTypeOf(f.s).toEqualTypeOf<Set<string>>();
expectTypeOf(f.t).toEqualTypeOf<WeakSet<SomeObj>>();
```

</details>

### Behavior

#### What does reading do?

Starting with a `{{ }}` block, we start reading values.

when reading a value created by `reactive`, (property on an object, item in a collection, "the value" (of a primitive, via current or via decorator)):
- entangle with the `{{ }}` (keep track of which reactive properties were accessed)

#### What does setting do?

- if a `{{ }}` block had previously entangled with a reactive property, the `{{ }}` will re-render.

### What are the performance concerns?

TODO

<!--

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

-->

## How we teach this

<!--

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

> Keep in mind the variety of learning materials: API docs, guides, blog posts, tutorials, etc.

-->

### Deep tracking

This type of tracking reactivity will lazily instrument all nested collections[^the-common-collections]

#### With primitive values

Template-only:
- return a [`Cell`](https://github.com/emberjs/rfcs/pull/1071)

```gjs
import { reactive } from '@ember/reactive';

const exclaim = (cell) => cell.current += '!';

<template>
    {{#let (reactive "hello") as |greeting|}}
        {{greeting.current}}

        <button {{on "click" (fn exclaim greeting)}}>Exclaim</button>
    {{/let}}
</template>
```

Class-based:
- creates a [`Cell`](https://github.com/emberjs/rfcs/pull/1071), since only decorators can intercept read/writes to primitive values

```gjs
import Component from '@glimmer/component';
import { reactive } from '@ember/reactive';

export class Demo extends Component {
    count = reactive(0);

    increment() {
        this.count.current++;
    }

    <template>
        {{this.count.current}}
        <button {{on "click" this.increment}}>++</button>
    </template>
}
```


#### Decorator + primitive value

Class based example, functionally the exact same as using `@tracked` today:

```gjs
import Component from '@glimmer/component';
import { reactive } from '@ember/reactive';

export class Demo extends Component {
    @reactive count = 0;

    increment() {
        this.count++;
    }

    <template>
        {{this.count}}
        <button {{on "click" this.increment}}>++</button>
    </template>
}
```

#### With the _common collections_[^the-common-collections]

Template-only:
- return an object matching the API and prototype of the passed in value 

```gjs
import { reactive } from '@ember/reactive';

const initialData = { greeting: 'hello' };
const exclaim = (data) => data.greeting += '!';

<template>
    {{#let (reactive initialData) as |data|}}
        {{data.greeting}}

        <button {{on "click" (fn exclaim data)}}>Exclaim</button>
    {{/let}}
</template>
```

Class-based:
- return an object matching the API and prototype of the passed in value 

```gjs
import Component from '@glimmer/component';
import { reactive } from '@ember/reactive';

export class Demo extends Component {
    data = reactive({ greeting: 'hello' });

    exclaim() {
        this.data.greeting += '!';
    }

    <template>
        {{this.data.greeting}}
        <button {{on "click" this.exclaim}}>Exclaim</button>
    </template>
}
```

### Shallow Tracking

This type of tracking reactivity will intrument one level of the common collection[^the-common-collections]

#### With the _common collections_[^the-common-collections]

Template-only:
- return an object matching the API and prototype of the passed in value 

```gjs
import { reactive } from '@ember/reactive';

const initialData = { greeting: 'hello' };
const exclaim = (data) => data.greeting += '!';

<template>
    {{#let (reactive.shallow initialData) as |data|}}
        {{data.greeting}}

        <button {{on "click" (fn exclaim data)}}>Exclaim</button>
    {{/let}}
</template>
```

Class-based:
- return an object matching the API and prototype of the passed in value 

```gjs
import Component from '@glimmer/component';
import { reactive } from '@ember/reactive';

export class Demo extends Component {
    data = reactive.shallow({ greeting: 'hello' });

    exclaim() {
        this.data.greeting += '!';
    }

    <template>
        {{this.data.greeting}}
        <button {{on "click" this.exclaim}}>Exclaim</button>
    </template>
}
```

## Drawbacks


<!--

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

-->

## Alternatives

<!--

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

-->

## Unresolved questions

<!--


> Optional, but suggested for first drafts. What parts of the design are still
TBD?

-->
