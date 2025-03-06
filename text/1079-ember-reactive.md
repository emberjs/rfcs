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


## Motivation

Example[^example-requires-vm-work]:
```gjs
import { reactive } from '@ember/reactive';

const startingData = { greeting: 'hi' };
const exclaim = (data) => data.greeting += '!'; 

export const Demo = <template>
    {{#let (reactive startingData) as |data|}}
        {{data.greeting}} === 'hi'
        <button {{on "click" (fn exclaim data)}}>Exclaim</button>
    {{/let}}
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

Feedback on prior RFCs, [The Cell](https://github.com/emberjs/rfcs/pull/1071), and [tracked-built-ins, built-in](https://github.com/emberjs/rfcs/pull/1068), revealed that folks have found that building reactive data structures, by default, is cumbersome. While there utilities would not be removed in anyway, we want to provide a way for folks to make things easily reactive, and _still_ provide a way to get fine-grained reactivity via the existing tools that folks are used to today.

The main tradeoff with deep reactivity vs explicit / shallow is the impact to memory. This RFC describes those tradeoffs, and provides a way for folks to determine when they may want to consider 

This doesn't eliminate or remove any existing APIs, but, being a low-level implementation would  

## Detailed design

Goals for `reactive`:
- decorator and non-decorator usage (compatible with typescript)
- handles both primitive values and the _common collections_[^the-common-collections]

[^the-common-collections]: The comment collection data structures are: `Array`, `Set`, `WeakSet`, `Object`, `Map`, `WeakMap`

### Proposed API / available usage

[Example Types at tsplay.dev](https://www.typescriptlang.org/play/?#code/JYWwDg9gTgLgBAbzgUwB5mQYxgFQJ4YDyAZnAL5zFQQhwDkaG2AtDAcnQNwBQ3A9HzgALGDDABnAFwCA5sBhCArgCMAdJhp9kIZcigArcXyjFMRsIoA2lvgEYADAHZbfYsEvJx3YADsYe4gBDTGQ4ACVkYJhgADdkAB4ANUDLRWQAPkRuODhMRSgoZD9JOGTU5B4cwsCAEwAKAEoSsrSeMl5ffxNg0IBhZGsklLTMtH8fGvFwyOxYhJaMrJyBOFU17nb+QTBLQN84RXFAmWRuYkUfWYgfOGrZuKHy9LrfCxhm4eQm0s+eFfEYMdQi4AExwGpYaCBGDQM4XK43O7ROJ1QFQE7vOAQZT6LAwAA0cAA1sg8CUAVBfDI4AAfODiPA6CCWQkQ8SYSlgGFQAD8JQuRJ8EAA7j5vgAFagYWB4AAikKg0Ogf0E4iYcEEOF6AGYAJxweUaRXcuGXaLXW4zZHzT7PNEYkq9XbicQAQUwIRd0ENUO5OEC6OQMHiAqFosJC3ShI0fjGjudbo9nnE3oVSqgvWu-lQMG+TsCLvdnpTUB9xugEXEVmDoZFPgjtpVcFAO20RUB5puqPY4Ignh8dHgIGhXUJaCiljwDVNCMtUTmjxGLx8bw+5W+CyWlpg+Ru9jgBYOPkFdYPUwWbV4WwEN9vt62cBwnhg4gAhNe75++N5Y90QnAAGUaGQQgcS3YgIAgEofEUHQ9EqOBlADaDkDiKBLwfbFcWwbhGDxfAiGIOokTmOokAgqC4DBMgGgaVQYQAWWhTAhFA7DcHYeJyMg6DYN0KByGeBomwDRU8Fw9B8PYEhiKtUiAG06AougAF1aPoiAAFEAEdFBSAiQOIeIKSpeSVKEpthzACSmA4wjZPnFEfGQYU4CYsBjJgSkfBkQkTJ8oT1JhHS9MsAySHidzPO83z6S8ql0gsh9hUiIk4CsmypPskinJcuAAHVUqioCQBAnFCWUSCPECHxAro4LdP06SjMKwIiWK4C2IqqrIlqpL-iDTLsHCoicuQOpnNcgCg2ihLGiCrTGrC5r4mm4N-JkRLGibFK2vpQa8OG5qHPucbJoK1K1tWzqcTqjSQqawj4laokrpKsr9C24TeBWCEjXTLxGGgeBMATOAADFIK3FZKoUOdTstYg9CKf8apqeHrTgGJPmyOAAAExtyCALngABeOB7AQgm5LiA84HJ7jKOoqnCeUem4EU5SVJZmnQkwdnzqija-PigLttx6nHNCdHyfOl6OtKrrEJ6mqkpySWEdCWW8quja1fxwnSG11yXrem7PvF3GVkJ7HyixHxJ0JQokcKS5QmAKYAWhYBMFx6lybGup7G+nIhHZwPGZKEEaIQ4Bw95upOcg1SQ7gfR46lia8qF0XYr1+bcbSgOE7lorAg896lcq5lesChDLAz06s6mmb89ohDaGLzPS7as3Fdu+aeCtwRGEsH35AxuZEeRt2DwmSfadttIJcJm4u6b4OealrFG+tMjKB4qiY5X3m4DAXeFKU5O1K3hHtIv3LXJzmKRZiwKT+3gT173nv2vL67+76G6jXVWg8P4IymN-Ui51da5zruAzGZMF5nTyqbGalcB7tw2LwDw8AjZwHOpDCAltDp2UMnUYg6hiZ+HqotUKI14gwTglAJKpCRoUNUIEWhD1lpPQQBRXizDBLizYcdShyhuFLQYRtMyrDJJHXspQzAkj6ErWflSV+CU5G2XYZQmoKjHqGWemXCu5tgHVT6iI+RZCZKUK+PdKRK1YFv20VlchlDiAGN4UYtBwYMGfSSkNGxRFKEyC8Qwph-FXEKPcaoIQ4SVr8MPpEvQwjvqiMUaoYACSnoyPMlYnRYjVD6ByUY9RPlNEBWicEjhRJSkRXlv-fx5ja7VN0aoSw9SjLOK0QUtxtjVAgC6cY3u6DzZfSHhk2JYoHGqKeiklhfSYkDIgMMpJlEFlpJ4FMgZYBhl5LaUU7Swzyl5zgYczJUBhmNNMYAlpqsLmxPEMMnpVSlk1MobmWZhiGmXTGYAiZvAgA)

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

// plain usage
function reactive<Value>(input: Value): Value;
// TC39 Stage 1 or 2 Decorator
function reactive(target: object, key: string | symbol, descriptor?: unknown): PropertyDecorator;
// TC39 Stage 3 or 4 Decorator (what is shipping to browsers)
function reactive<Value>(target: ClassAccessorDecoratorTarget<unknown, Value>, context: ClassAccessorDecoratorContext): ClassAccessorDecoratorResult<unknown, Value>;
// implementation (type doesn't matter, exactly)
function reactive<Value>(input: Value): Value {
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
  g = reactive(0);
  h = reactive({ foo: 2});
  i = reactive(['foo']);
  j = reactive(new Map<string, string>())
  k = reactive(new WeakMap<SomeObj, boolean>());
  l = reactive(new Set<string>());
  m = reactive(new WeakSet<SomeObj>());

  // explicit reactive reference and reactive value
  @reactive n = reactive(0);
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

expectTypeOf(f.g).toEqualTypeOf<number>();
expectTypeOf(f.h).toEqualTypeOf<{foo: number }>();
expectTypeOf(f.i).toEqualTypeOf<string[]>();
expectTypeOf(f.j).toEqualTypeOf<Map<string, string>>();
expectTypeOf(f.k).toEqualTypeOf<WeakMap<SomeObj, boolean>>();
expectTypeOf(f.l).toEqualTypeOf<Set<string>>();
expectTypeOf(f.m).toEqualTypeOf<WeakSet<SomeObj>>();

expectTypeOf(f.n).toEqualTypeOf<number>();
expectTypeOf(f.o).toEqualTypeOf<{foo: number }>();
expectTypeOf(f.p).toEqualTypeOf<string[]>();
expectTypeOf(f.q).toEqualTypeOf<Map<string, string>>();
expectTypeOf(f.r).toEqualTypeOf<WeakMap<SomeObj, boolean>>();
expectTypeOf(f.s).toEqualTypeOf<Set<string>>();
expectTypeOf(f.t).toEqualTypeOf<WeakSet<SomeObj>>();
```

</details>

#### With primitive values

```gjs
    
```

#### Decorator + primitive value

#### With the _common collections_[^the-common-collections]



### Behavior

#### What does reading do?

#### What does setting do?

### What are the performance concerns?

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
