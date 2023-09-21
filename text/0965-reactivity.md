---
stage: accepted
start-date: 2023-09-20T00:00:00.000Z # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/965 
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

# Rethinking Reactivity

## Summary

Leading up to the Octane Edition, we laid out a plan to get rid of `<object>.get` and `<object>.set' in favor of `@tracked` class properties, which paved the way for writing reactive code in a way that is _the most_ "just javascript" out of our framework peers at the time,

Additionally, this was taken as an opportunity to move from push-based reactivity to pull-based reactivity, and lead to a long exploration of patterns, trial-and-error, and developing new patterns to achieve easier testing, debugging and maintenance.  But most of this happened long after the official documentation for Octane had landed -- and even at the time of writing this RFC, the most important concepts about reactivity would fall well under the ["In-depth topics"](https://guides.emberjs.com/release/in-depth-topics/autotracking-in-depth/) section of the documentation. 

This RFC aims for two things:
- focus on teaching reactivity concepts, so that folks can be better equipped for what would be "advanced concepts" today.
- defining and brining cohesion to our reactive primitives so that the above is much easier to archive. 
  - This may necessitate additional sub-RFCs. 

## Motivation

The ember ecosystem has a number of "third-party" abstractions for some core reactive primitives, that could benefit (from teaching, performance, and cohesion perspectives) from being pulled in to a cohesive set of utilities from one source. This will help with longer-term goals of [simplifying imports](https://github.com/emberjs/rfcs/pull/946) -- today we have a _pit of exhaustion_ when it comes to how many imports are needed to make components.


We have an opportunity to improve a lot of performance in how ember renders by collapsing abstractions. 

The "third-party" libraries are:
- `ember-modifier`, providing: `modifier`
- `ember-resources`, providing: `resource`, `cell`
- `tracked-built-ins`, providing: `TrackedArray`, `TrackedObject`, etc 

There are others, but these are the most commonly used. `@glimmer/*` was meant to be the _foundation_ upon which ember was built, and it still provides that function. Since we've introduced modifier managers, and helper managers, we've enabled a healthy amount of experimentation, and it's time to reign it all in under a cohesive story.


We want to distill concepts so that folks coming from other ecosystems, can more easily adopt, and learn one thing at a time as fast or as slow as they want.

<detail><summary>two things not mentioned here</summary>

1. helpers - since ember-source 4.5 (3.25 with the [polyfill](https://github.com/ember-polyfills/ember-functions-as-helper-polyfill)), helpers are a concept of the past. We can use regular functions -- 0 user-land abstraction.  However, changes to the default helper manager will be required to support `modifier` and `resource`, when used in a certain way (preview: this will be called "collapsing")

2. components - components are not a primitive, and moreso a refactoring boundary -- they combine the above things into packaged micro-bundles.

</details>

## Detailed design

tl;dr:
- define primitives
- define integration points
- manager pattern is fallback, rather than default

[starbeam]: https://www.starbeamjs.com/

### Preface

Before getting in to details of what **each thing is, how it works, and how we'll integrate and support it**, we need to go over some _equivalence_, because there is some overlap between [Starbeam][starbeam] and exports from `ember-modifier` and `ember-resources`. 

**All of the following are conceptually equivalent**

<details><summary>Values</summary>


[s-cells]: https://www.starbeamjs.com/guides/fundamentals/cells.html
[e-cells]: https://ember-resources.pages.dev/funcs/cell

In Starbeam, values are [Cells][s-cells] 

```js
import { Cell } from "@starbeam/universal";
 
const num = Cell(0);
expect(num.current).toBe(0);
```

In ember-resources, the `Cell` from Starbeam is implemented as [`cell`][e-cells]:

```js
import { cell } from "ember-resources";
 
const num = cell(0);
expect(num.current).toBe(0);
```

In `@glimmer/tracking`, `@tracked` can be thought of as a wrapper of `cell`

Specifically, `@tracked` could be implemented as:
```js
class Demo {
  #num = cell(0);

  get num() {
    return this.#num.current;
  }
  set num(value) {
    this.#num.current = value;
  }
}
```
which would be the exact same as what folks are used to today in Ember:
```js
import { tracked } from '@glimmer/tracking';

class Demo {
  @tracked num = 0;
}
```

Note, however, that with decorators landing as a platform feature, 
we'd use this, long-term:
```js
import { tracked } from '@glimmer/tracking';

class Demo {
  @tracked accessor num = 0;
}
```

Also note that originally, the cell API was motivated by and being a way to 
use auto-tracking without a compile step (as the stage-1-style decorators that 
ember folks are used to require compilation).

</details>
<details><summary>Resources</summary>

In Starbeam, a resource looks like this: 
```js
import { Cell, Resource } from "@starbeam/universal";
 
export const Now = Resource(({ on }) => {
  const now = Cell(Date.now());
 
  const timer = setInterval(() => {
    now.set(Date.now());
  });
 
  on.cleanup(() => {
    clearInterval(timer);
  });
 
  return now;
});
```

and in ember-resources, it looks like this:
```js
import { cell, resource } from "ember-resources";
 
export const Now = resource(({ on }) => {
  const now = cell(Date.now());
 
  const timer = setInterval(() => {
    now.set(Date.now());
  });
 
  on.cleanup(() => {
    clearInterval(timer);
  });
 
  return now;
});
```

They are nearly identical.

Where they differ mostly, however, is the pattern for building Resources.

For example, if we need to pass arguments to configure a resource, starbeam would look like this:
```js
import { Cell, Resource } from "@starbeam/universal";
 
export function Now(updateMs) {
    return Resource(({ on }) => {
        const now = Cell(Date.now());

        const timer = setInterval(() => {
            now.set(Date.now());
        }, updateMs);

        on.cleanup(() => {
            clearInterval(timer);
        });

        return now;
    });
}
```

and while we _could_ make the exact same thing using `ember-resources` with just changing the import and casing of `Cell` -> `cell` and `Resource` -> `resource`, it's _insufficient_ if we want to treat rendering resources in the template as equal-importance as usage in JavaScript.

```js
import { cell, resource } from "ember-resources";
 
 // works as you'd expect in JS, doesn't finish evaluating when invoked from a template
export function Now(updateMs) {
    return resource(({ on }) => {
        const now = cell(Date.now());

        const timer = setInterval(() => {
            now.set(Date.now());
        }, updateMs);

        on.cleanup(() => {
            clearInterval(timer);
        });

        return now;
    });
}
```

to remedy this, there is a utility that _would be_ a no-op, if it didn't register a custom _helper manager_ to handle the "collapsing" of the invocations:

```js
import { cell, resource, resourceFactory } from "ember-resources";
 
 // works as you'd expect in JS, doesn't finish evaluating when invoked from a template
export const Now = resourceFactory((updateMs) => {
    return resource(({ on }) => {
        const now = cell(Date.now());

        const timer = setInterval(() => {
            now.set(Date.now());
        }, updateMs);

        on.cleanup(() => {
            clearInterval(timer);
        });

        return now;
    });
});
```

It turns out that this `resourceFactory` concept _juuuust_ crosses the threshold of comfort when folks are learning resources in general. 

Later on in the RFC when "collapsing" is covered, we'll be able to remove reliance on `resourceFactory`.


</details>
<details><summary>Modifiers</summary>

In Starbeam, modifiers are _a pattern_, rather than anything specific, 

```js 
import { Resource } from "@starbeam/universal";

function Wiggle(element) {
  return Resource(({ on }) => {
    let frame;

    let randomTranslate = () => {
      element.animate(...);
      frame = requestAnimationFrame(randomTranslate);
    }

    frame = requestAnimationFrame(randomTranslate);

    on.cleanup(() => cancelAnimationFrame(frame));
  });
}
```

In ember-resources, a modifier is about the same,
with the caveat that a `modifier` wrapper invocation call is needed 
due to needing to register the passed function with a _modifier manager_.

Later on in the RFC when "collapsing" is covered, we'll be able to remove reliance on `modifier`.

```js 
import { resource } from 'ember-resources';
import { modifier } from 'ember-resources/modifier';

const Wiggle = modifier(function Wiggle(element) {
  return resource(({ on }) => {
    let frame;

    let randomTranslate = () => {
      element.animate(...);
      frame = requestAnimationFrame(randomTranslate);
    }

    frame = requestAnimationFrame(randomTranslate);

    on.cleanup(() => cancelAnimationFrame(frame));
  });
});
```

In ember-modifier, a modifier is an arrow function registered with a _modifier manager_
for each created modifier. 

```js 
import { modifier } from 'ember-modifier';

const Wiggle = modifier((element) => {
    let frame;

    let randomTranslate = () => {
      element.animate(...);
      frame = requestAnimationFrame(randomTranslate);
    }

    frame = requestAnimationFrame(randomTranslate);

    return () => cancelAnimationFrame(frame);
});
```

This is the _least_ typing of the options, and it make make sense to keep this API 
in the future, even if it's a wrapper around the resource concept.

For example: 

```js
function modifier(builder) {
  return (element, ...args) => {
    return resource(({ on }) => {
      let maybeDestructor = builder(element, ...args);

      if (maybeDestructor) {
        on.cleanup(maybeDestructor);
      }
    })
  };
}
```


</details>

### The Primitives 

- cells, values
- functions
- resources
- modifiers

### Un-primitiving the Primitives

- `@tracked`, etc

### The Utils

- `TrackedArray`
- `TrackedObject`
- `TrackedSet`
- `TrackedMap`
- `TrackedWeakMap`
- `TrackedWeakSet`
- etc?

Note: performance issues, incomplete implementation as ecma advances

### Invocation Collapsing


## How we teach this

### New section in the guides

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

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
