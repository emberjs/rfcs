---
stage: accepted
start-date: 2025-01-11T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - framework
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
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

# Add a Utility for Ownership Linkage 

## Summary

This RFC Proposes a new built-in utility to make ownership and destroyable/lifetime linkage easier.


## Motivation

Today, when we want to use "native classes", but still have the `owner` and destroyable linking, we must correctly incant the following:

```js
import Component from '@glimmer/component';
import { cached } from '@glimmer/tracking';
import { getOwner, setOwner } from '@ember/owner';
import { associateDestroyableChild } from '@ember/destroyable';

class MyClass { /* ... */ }

export default class Demo extends Component {
  @cached 
  get myInstance() {
    let instance = new MyClass();
    
    associateDestroyableChild(this, instance);
    
    let owner = getOwner(this);

    if (owner) {
      setOwner(instance, owner);
    }

    return instance;
  }
}
```

In applications where developers want to emphacise using "plain javascript" and abstracting _away_ from emberisms, this marathon of boilerplate can feel very cumbersome. 

Instead, the above could be done in a single utility, since linkage is always the same:

```js
import Component from '@glimmer/component';
import { link } from '@ember/lifetime';

class MyClass { /* ... */ }

export default class Demo extends Component {
    @link(MyClass) declare myInstance: MyClass;
}
```

Or in the case where you may need arguments, you can always go back to using `@cached` getters, which will correctly entangle with tracked properties, and re-create the class if needed.
```js
import Component from '@glimmer/component';
import { cached } from '@glimmer/tracking';
import { link } from '@ember/lifetime';

class MyClass { /* ... */ }

export default class Demo extends Component {
  @cached 
  get myInstance() {
    // changes to foo cause another MyClass to be created
    let instance = new MyClass(this.args.foo);
    
    link(this, instance);

    return instance;
  }
}
```
Note however, that if `MyClass` implemented or needed destruction, this would leak memory until the whole component is torn down. For fine-grained, per-property lifetimes, Resources would be a better fit.
(Resources' implementation would benefit from the `link` utilty as well, as an implementation detail)

This could also be done inline, assuming new owner-access is needed during construction:
```ts
import Component from '@glimmer/component';
import { cached } from '@glimmer/tracking';
import { link } from '@ember/lifetime';

class MyClass {
    constructor(fooFn) {
        this.#fooFn = fooFn;
    }

    // Lazy access is reactive access
    get foo() {
        return this.#fooFn();
    }
}

export default class Demo extends Component {
  @link myInstance = new MyClass(() => this.args.foo);
}
```



## Detailed design

tl;dr: add `link` to a new package oriented around "lifetimes": `@ember/lifetime`.

The implementation of this has already existed for some time in a community library, with tests, etc, and is copied below:

~~~ts
import { getOwner, setOwner } from '@ember/application';
import { assert } from '@ember/debug';
import { associateDestroyableChild } from '@ember/destroyable';

// While this implementation is for Stage1 Decorators,
// the real implementation, should this RFC be accepted,
// should support spec-decorators as well.
import type { Class, Stage1Decorator, Stage1DecoratorDescriptor } from '#types';

type NonKey<K> = K extends string ? never : K extends symbol ? never : K;

/**
 * A util to abstract away the boilerplate of linking of "things" with an owner
 * and making them destroyable.
 *
 * ```js
 * import Component from '@glimmer/component';
 * import { link } from 'reactiveweb/link';
 *
 * class MyClass {  ... }
 *
 * export default class Demo extends Component {
 *   @link(MyClass) myInstance;
 * }
 * ```
 */
export function link<Instance>(child: Class<Instance>): Stage1Decorator;

/**
 * A util to abstract away the boilerplate of linking of "things" with an owner
 * and making them destroyable.
 *
 * ```js
 * import Component from '@glimmer/component';
 * import { cached } from '@glimmer/tracking';
 * import { link } from 'reactiveweb/link';
 *
 * export default class Demo extends Component {
 *   @cached
 *   get myFunction() {
 *     let instance = new MyClass(this.args.foo);
 *
 *     return link(instance, this);
 *   }
 * }
 * ```
 *
 * NOTE: If args change, as in this example, memory pressure will increase,
 *       as the linked instance will be held on to until the host object is destroyed.
 */
export function link<Child, Other>(child: Child, parent: NonKey<Other>): Child;

/**
 * A util to abstract away the boilerplate of linking of "things" with an owner
 * and making them destroyable.
 *
 * ```js
 * import Component from '@glimmer/component';
 * import { link } from 'reactiveweb/link';
 *
 * class MyClass {  ... }
 *
 * export default class Demo extends Component {
 *   @link myInstance = new MyClass();
 * }
 * ```
 *
 * NOTE: reactive args may not be passed to `MyClass` directly if you wish updates to be observed.
 *   A way to use reactive args is this:
 *
 * ```js
 * import Component from '@glimmer/component';
 * import { tracked } from '@glimmer/tracking';
 * import { link } from 'reactiveweb/link';
 *
 * class MyClass {  ... }
 *
 * export default class Demo extends Component {
 *   @tracked foo = 'bar';
 *
 *   @link myInstance = new MyClass({
 *      foo: () => this.args.foo,
 *      bar: () => this.bar,
 *   });
 * }
 * ```
 *
 * This way, whenever foo() or bar() is invoked within `MyClass`,
 * only the thing that does that invocation will become entangled with the tracked data
 * referenced within those functions.
 */
export function link(...args: Parameters<Stage1Decorator>): void;

export function link(...args: any[]) {
  if (args.length === 3) {
    /**
     * Uses initializer to get the child
     */
    return linkDecorator(...(args as Parameters<Stage1Decorator>));
  }

  if (args.length === 1) {
    return linkDecoratorFactory(...(args as unknown as [any]));
  }

  // Because TS types assume property decorators might not have a descriptor,
  // we have to cast....
  return directLink(...(args as unknown as [object, object]));
}

function directLink(child: object, parent: object) {
  associateDestroyableChild(parent, child);

  let owner = getOwner(parent);

  if (owner) {
    setOwner(child, owner);
  }

  return child;
}

function linkDecoratorFactory(child: Class<unknown>) {
  return function decoratorPrep(...args: Parameters<Stage1Decorator>) {
    return linkDecorator(...args, child);
  };
}

function linkDecorator(
  _prototype: object,
  key: string | symbol,
  descriptor: Stage1DecoratorDescriptor | undefined,
  explicitChild?: Class<unknown>
): void {
  assert(`@link is a stage 1 decorator, and requires a descriptor`, descriptor);
  assert(`@link can only be used with string-keys`, typeof key === 'string');

  let { initializer } = descriptor;

  assert(
    `@link requires an initializer or be used as a decorator factory (\`@link(...))\`). For example, ` +
      `\`@link foo = new MyClass();\` or \`@link(MyClass) foo;\``,
    initializer || explicitChild
  );

  let caches = new WeakMap<object, any>();

  return {
    get(this: object) {
      let child = caches.get(this);

      if (!child) {
        if (initializer) {
          child = initializer.call(this);
        }

        if (explicitChild) {
          // How do you narrow this to a constructor?
          child = new explicitChild();
        }

        assert(`Failed to create child instance.`, child);

        associateDestroyableChild(this, child);

        let owner = getOwner(this);

        assert(`Owner was not present on parent. Is instance of ${this.constructor.name}`, owner);

        setOwner(child, owner);

        caches.set(this, child);
        assert(`Failed to create cache`, child);
      }

      return child;
    },
    // TS doesn't understand Stage 1 decorators
  } as unknown as void /* Thanks TS. */;
}
~~~

## How we teach this

- API Docs above should cover most of it
- In "In-Depth Topics" on the guides, in "Native Classes In-Depth", we should add a section for linking. This currently isn't covered at all.

The text could read something like this: 

### Linking Lifetimes

> [!NOTE]
> We probably need another section somewhere describing some concepts. Lifetimes have existed in ember forever, but we've never described what they are explicitly anywhere.

Starting from any framework-owned instance, a plain JavaScript object, class, (etc), can be linked, providing the owner and lifetime linkage to that plain JavaScript object, class, etc.

This means, that the plain JavaScript object, class, etc, will be able to inject services, and registerDestructors (with a link to the registerDestructor api).

For example, say you have a utility class: 

```js
export class MyClass {
    @service router;

    get queryParams() {
        return this.router.currentRouter?.queryParams ?? {};
    }
}
```

In order for the `@service` injection to work, the creation of this class needs to include linkage, which can be done from any framework-owned instance, or any other instance that already has ownership linkage, such as in a service:

```js
import Service, { service } from '@ember/service';
import { link } from '@ember/lifetime';

export default class MyService extends Service {
   @link(MyClass) myClass; 

    get qps() {
        // works!
        return this.myClass.queryParams;
    }
}
```

You can also setup destructors in the constructor, like this:

```js
export class MyClass {
    constructor() {
        window.addEventListener('resize', this.greet);

        registerDestructor(this, () => {
            window.removeEventListener('resize', this.greet);
        });
    }

    greet = () => console.log('hi');
}
```

Destruction helps prevent memory leaks both in running apps, and in tests.


Note, however, if a linked object is re-created during a component's lifetime, prior linked objects will not have been cleaned on that component until the component is torn down.

> [!NOTE]
> We don't talk about lazy argument passing anywhere in the guides, and that would help a ton for linkage documentation, and may be a prereq for documenting this -- but should not block this RFC as the concepts already exist and are in use today, but just need to be documented.. There are many use cases for wanting plain classes with reactivity without using framework super-classes (services, components, etc) -- such as utilities, high-level abstractions (linking query params to local storage, having pre/post-processing of data wrapped, working with graphics, canvas, animations, etc). But in this section about argument passing, we'd need to talk about lazy evaluation without needing cleanup, and how without cleanup, we would end up needing resources, and those have not been RFC'd yet.


## Drawbacks

- Another API (I think there are more drawbacks to not doing this tho)

## Alternatives

- Different import path 
- Different name

## Unresolved questions

- n/a
