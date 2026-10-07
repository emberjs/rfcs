---
Stage: Accepted
Start Date: 2021-06-15
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/753
---

# Logging and Debugging Context API

## Summary

Adds an API that can be used to provide logging and debugging information in
both development and production builds:

```js
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { getDebuggingContext } from '@glimmer/debug';
import { action } from '@ember/object';

export default class MyComponent extends Component {
  @tracked data;

  @action
  async fetchData() {
    try {
      let response = await fetch(this.args.url);
      this.data = await response.json();
    } catch (e) {
      let context = getDebuggingContext(this).join('\n');

      console.log(`Fetch request failed! This occured in:\n\n ${context}`);
    }
  }
}
```

## Motivation

Currently, it can be pretty difficult to understand what caused an error within
an Ember application. Errors can occur almost anywhere at any time. They can be
thrown while rendering a template, or within an action, or in an when a request
fails. They can be in a Backburner queue, or in a chain of promises, or in an
event handler. The asynchronous nature of browser apps makes tracking down
errors and figuring out where they occured more of an art form than a science
at times. This is doubly true for production applications, which remove much of
Ember's assertion logic that provides helpful debugging context, at the cost of
speed.

The number one piece of information that can help developers to figure out where
a bug is coming from is what component it came from, and specifically, in which
part of the component _tree_. Doing this narrows down the scope of a bug
massively, because not only do we now know where the offending code resides, we
also roughly know which _instance_ of the component it resides within. This is
similar to seeing a stack trace which shows us not only the function where an
error occured, but all of the functions that called that function as well.
Without this context, it would be difficult to use a standard debugger, and
similarly without the full context of the component tree is is difficult to
debug Ember apps.

This RFC outlines a new API for providing this context to developers in
development and in production builds of Ember apps. The context will be
generated on demand, so it will not incur additional costs when it is not used.

To do this, we will utilize the _destroyable tree_. This tree was introduced in
the [destroyables RFC](https://github.com/emberjs/rfcs/blob/master/text/0580-destroyables.md),
and is used by the VM to manage the lifecycle of components, helpers, modifiers,
and other constructs. Child components are registered on parents in this tree,
so it roughly reflects the structure of the component tree, which is exactly
what we want here. And because destruction can occur at any point, children
maintain a link to their parents so they can de-register themselves upon
destruction. We can use this link to crawl backward from a given destroyable
through its parents on demand, building up the debugging context as we do.

Since we are utilizing an existing datastructure which _necessarily_ exists in
production apps, we won't be introducing any additional costs by adding this
API. We also won't need to retrofit this new API into the VM, as the
datastructure already exists. In addition, because users can add onto the
destroyable tree themselves, this API can also be used to provide context for
user constructs, making it extensible.

## Detailed design

As discussed in the motivation section, this API will build on the destroyables
API, using it as the basis for how it operates. We will introduce the following
new functions:

```ts
interface PartialContextDescriptor {
  description: string;
  extra?: unknown;
}

interface ContextDescriptor extends PartialContextDescriptor {
  parent: ContextDescriptor | ContextDescriptor[] | null;
  isDestroying: boolean;
  isDestroyed: boolean;
}

declare function getDebuggingContext(obj: object): ContextDescriptor | null;
declare function setDebuggingContextGenerator(
  obj: object,
  generator: (obj: object) => PartialContextDescriptor
): void;
```

These functions will be importable from the `@glimmer/debug` module, since they
have to do with debugging. Unlike other debugging functions, like the ones
provided in `@ember/debug`, these will _not_ be removed from applications in
production builds since they will still provide useful behavior, and will not
incur performance penalties until they are used.

### getDebuggingContext

`getDebuggingContext` is passed an object, and returns either a descriptor of
that object which can be used for debugging purposes, or `null` if the object
does not have any debugging context generators associated with it or any of its
parents. The debugging context object contains the following properties:

- `description`: A string which describes the object itself
- `parent`: If the object has a parent (see below) which also has a debugging
  context, then this property is that context object. If there are multiple
  such parents, then it is an array of objects. Otherwise, it is `null`.

Debugging context parents are the same parents that are associated with the
object via `associateDestroyableChild`. If a particular parent exists, but does
not have a debugging context generator, then it is skipped and its parent(s) are
used instead.

Debugging context generators for a given object are set via
`setDebuggingContextGenerator`. When you call `getDebuggingContext` on a
particular object, it first checks to see if a generator was associated directly
with that object. If one wasn't, it then checks the prototypes of that object.
This way, debugging context generators can be associated with classes instead of
individual objects. When one is found, it is called and passed the object that
`getDebuggingContext` was called on.

### setDebuggingContextGenerator

`setDebuggingContextGenerator` associates a generator function with a given
object. This generator function is called when `getDebuggingContext` is called
on that object, or if the object is a class, then when it is called on an
instance of the class. The generator is inherited, so subclasses will get the
same generator unless it is overridden.

The generator function receives the object itself as its only parameter. It
should return an object with the following properties:

- `description`: A helpful string describing the object for debugging purposes.
- `extra`: This is an optional property which can contain extra debugging
  information. It can be any type of value.

## How we teach this

This API is meant to be used by library authors and framework authors, so it
does not need a dedicated section in the guides yet. In time, a section could be
added in the In-Depth Topics section once best practices are established.

In the meantime, the descriptions of the new functions from the detailed design
section above can be used as the basis for API documentation.

## Drawbacks

The main drawback is that this API cements the need for a child-to-parent link
in destroyables, as it is necessary to be able to find the full debugging
context at any time for a given object. This was not specified directly by the
previous RFC, but it is necessary to prevent memory leaks, since children can be
destroyed at any time, and if they are not removed from their parents then they
will leak.

## Alternatives

- We could use a different import path than `@glimmer/debug`. This could be
  useful, as we may want to reserve that namespace for APIs like `assert` and
  `deprecate` which _will_ be stripped from production, and co-mingling stripped
  and non-stripped APIs could be confusing.

- We could use a separate tree specifically for debugging context/information.
  This would likely be very costly from a performance perspective, and would
  likely be prohibitive.

- We could only ship this API in development and not production, relaxing the
  performance constraints. While this would give us more flexibility in design,
  production bugs are currently some of the most difficult to debug in Ember
  today, and providing context for production is a crucial part of this design.
