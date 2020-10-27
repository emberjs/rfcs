- Start Date: 2020-10-27
- Relevant Team(s): Ember.js, Learning (?)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Documenting Components

## Summary

This RFC describes a way to document components in a way to connect it to tooling from the wider javascript ecosystem. The scope of this RFC is to cover documentation for **arguments**, **named blocks** and **yield** for components written in JavaScript, TypeScript and template-only.

## Motivation

The current preferred way to document your components is probably using [ember-cli-addon-docs](https://github.com/ember-learn/ember-cli-addon-docs), trying to use [storybook](https://storybook.js.org/), _older_ ember addons (such as [ember-freestyle](https://github.com/chrislopresto/ember-freestyle)), one of the [empress](https://github.com/empress) products or running your own system. The way these libraries extract information from your code is mostly in a _proprietary_ format (eg. `ember-cli-addon-docs` maintains its own standard and system of generating such documentation). For `ember-cli-addon-docs` this has proven to be volatile and maintenance is part of the respective projects. 

The idea is to find a documentation standard that connects with the rest of javascript ecosystem rather than building one on its own and is recommended for usage within the ember community.

### Example: `<Modal>` Component

Let's start with an example to document a **template-only** `<Modal>` component with arguments and optional named blocks (and their yielded properties):

```ts
// addon/components/modal/index.d.ts
import Component, { Template } from '@glimmer/component';

export interface ModalArgs {
  /** Trigger the visibility of the modal */
  open: boolean;
  /** Callback when the modal was opened */
  opened: () => void;
  /** Callback when the modal was closed */
  closed: () => void;
}

export interface ModalBlocks {
  default: [close: () => void];
  header: [close: () => void];
  body: [];
  footer?: [];
}

export default class ModalComponent extends Component<ModalArgs, ModalBlocks> {}
```

which in turn we can use in these two variants (with and without named blocks):

```hbs
<Modal as |close|>
  Here is the content of the modal
  
  <button {{on "click" close}}>close</button>
</Modal>

<Modal @open={{true}}>
  <:header as |close|>
    This is my title
    
    <button {{on "click" close}}>close</button>  
  </:header>
  
  <:body>
    Here is the content of the modal
  </:body>
</Modal>
```

## Detailed design

The design is sectioned into _definitions_, _documentation API_, _examples_ and then summarized with _benefits_.

### Definitions

At first let's define terms that will be used throughout this RFC:

- **Block**
  For any block invocation of a component, this describes the contents in there. We can distinguish between _default_ block and named blocks.
  This is defined in syntax section for [RFC 460 - Yieldable Named Blocks](https://emberjs.github.io/rfcs/0460-yieldable-named-blocks.html#syntax)
  
  - _Default Block_
    That is the default block when invoking a component:

    ```hbs
    <Modal>
      this is the default block
    </Modal>
    ```
    
    equivalent to:
    
    ```hbs
    <Modal>
      <:default>this is the default block</:default>
    </Modal>
    ```

  - _Named Block_
    Named blocks can be referred to via an identifier and are represented through `<:identifier>` syntax:
 
    ```hbs
    <Modal>
      <:header>Named header block</:header>
      <:body>Named body block</:body>
    </Modal>
    ```
- **Block Param**
  Parameters passed from the component (through `{{yield}}`) to the consumer:
  
  ```hbs
  <Modal as |close|>
    <button {{on "click" close}}>close</button>
  </Modal>
  ```
  
  In the example `close` is a parameter _to_ the block. On the consuming side a space delimited list of parameters can be accessed within the pipe syntax (`| .. |`).
  
- **Yield**
  Yielding is the mechanism from _inside_ the component to provide named blocks and pass parameters to them. Examples from RFC 460:
  
  ```hbs
  <article>
    <header>{{yield @article.title to='header'}}</header>
    <section>{{yield @article.body to='body'}}</section>
  </article>
  ```
  
### Documentation API

Documentation will use the powerful typing system of TypeScript. For TypeScript this works out of the box. For template-only components, use a `my-component.d.ts` file. For JavaScript, you can choose between a `*.d.ts` file or jsdoc with extended tsdoc annotations (see examples below).

The interface for `@glimmer/component` needs extension for yields:

```ts
export default MyComponent extends Component<Input, Output> {}
```

The narrative is to give developers documentation slots for `Input` and `Output`. The `Input` will continue to work as is to document arguments and `Output` will be added to document blocks with their parameters.

```ts
interface Blocks {
  [block: string | 'default']: []; // the value will be a tuple
}

type Output = Blocks; // only here to explain the narrative

export default class Component<Input = {}, Output = Blocks> { }
```

1. With named blocks: When a component uses named blocks, then each block must be named in the `Output` interface. Blocks can be marked optional with the `?` sigil as suffix.
2. Without named blocks: Use `default` to describe the _default_ block of a component.

The `Output` bridges the yield mechanics (from the perspective of the developer of a component) to block consumption mechanics (from the perspective of a consumer of this component). A consumer can access either parameters of the default block (`<Modal as |close|>`) or the parameters of named blocks (`<:header as |close|>`) in their declarative invocation but never both at the same time.

**This will require to change the types in `@glimmer/component` package**

### Example for Parameters of the Default Block: `<BlogPost>` Component

The type to each block is a function and the parameters represent the order of the yielded properties. Here is the [blog post example](https://guides.emberjs.com/release/components/block-content/#toc_block-parameters) from ember guides:

```ts
// typescript

interface BlogPostArgs {
  post: {
    title: string;
    author: string;
    body: string;
  };
}

interface BlogPostBlocks {
  default: [title: string, author: string, body: string];
}

export default class BlogPostComponent extends Component<BlogPostArgs, BlogPostBlocks> {}


// javascript

/**
 * @typedef {object} BlogPostArgs
 * @property {object} post
 * @property {string} post.title
 * @property {string} post.author
 * @property {string} post.body
 */

/**
 * @typedef {{default: [title: string, author: string, body: string] }} BlogPostBlocks
 */

/**
 * @extends {Component<BlogPostArgs, BlogPostBlocks>}
 */
export default class BlogPostComponent extends Component {}
```

### Example for the Default Block with an Object Parameter: `<InputBuilder>` Component

A param can of course be a object-hash as this is propably the most-often use-case. Here is an `<InputBuilder>` component, yielding a couple of CSS classes ready to construct yours:

```ts
// typescript

interface Parts {
  affixClass: string;
  prefixClass: string;
  suffixClass: string;
}

interface InputBuilderBlocks {
  default: [parts: Parts];
}

export default class InputBuilderComponent extends Component<{}, InputBuilderBlocks> {}


// javascript

/**
 * @typedef {object} Parts
 * @prop {string} affixClass
 * @prop {string} prefixClass
 * @prop {string} suffixClass
 */

/**
 * @typedef {object} InputBuilderBlocks
 * @prop {[parts: Parts]} default
 */

/**
 * @extends{Component<{}, InputBuilderBlocks>}
 */
export default class InputBuilderComponent extends Component {}
```

### Benefits

To summarize the benefits of this approach

- Powerful type system to document components
- Integration with the wider javascript ecosystem (e.g. [api-extractor](https://api-extractor.com))
- Integration with tooling, such as language server for intelli-sense
- NOT an isolated ember only solution
- For javascript _and_ typescript

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

Extend the ember guides with the terminology of this RFC. Update the documentation and explain how to use this to document your components (probably under an advanced section?)

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

Additions on "how to document your component" and explain terminology around yield.

> How should this feature be introduced and taught to existing Ember
users?

As RFCs are nowadays very well announced through ember-times, that is enough.

Ideally, this should enable the community to probe solutions and figure out a process to standardize this even further, such as providing a generic access to a metadata file that can be consumed by documentation tools.

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

Maintaining own documentation standard has been proven to be impractical as part of `ember-cli-addon-docs` and isolates ember from the wider javascript ecosystem. Instead it has proven to join forces on particular topics and strive towards solutions evolved with the javascript community that work with ember and other frameworks.

> There are tradeoffs to choosing any path, please attempt to identify them here.

This RFC may be read or understand as a way to opt-in to typescript, yet isn't. Even developers using javascript only as of today benefit from tooling that is built using typescript and now they can utilize this for themselves.

(The grammar for docblocks is influenced by [tsdoc](https://github.com/microsoft/tsdoc) plus the offerings from typescript's type system.)

## Alternatives

### Alternative Technologies
There are alternatives that have been used for documentation purposes, too:

- **JS(doc) dialect/schema** - This is the approach of current `ember-cli--addon-docs`. Most of the times this is jsdoc with support for ember specific doc-tags (e.g. @argument or @yield). The type support for either of those is pretty much what you choose it to be, hardly consumed by third parties and rendered on `ember-cli-addon-docs` "as is". I think this approach is limited by technology and subjectively feels very much like free text.

- **JSONSchema** - There is a way in bending its use to document components. Similar to the approach above it then needs ember specific keywords and tools to understand, postprocess and display that.

Both of these alternatives share the problem, that they give you a format in which you can document the needed information but both of these are then not understood. Both would require post-processing by special tools to give the information the required meaning. They wouldn't reach the benefits of the outline approach here.

### Alternative Typings

Also discussed was the typing of blocks. An earlier draft considered functions to match blocks as invocables, like so:

```ts
type Template = unknown;

export interface ModalArgs {...}

export interface ModalYield {
  (close: () => void): Template;
  header: (close: () => void) => Template;
  body: () => Template;
  footer?: () => Template;
}

export default class ModalComponent extends Component<ModalArgs, ModalYield> {}
```

Despite of its charmant usage of [function types](https://www.typescriptlang.org/docs/handbook/interfaces.html#function-types) for the _default_ block, the return value couldn't be properly typed. That goes back to some earlier experiments by Dan Freeman. The return value of that block would return the template, but given the template may have `{{yield}}` statements, the return value would be something more of a _generator_. Second with the advent of tuples, they better synchronize the developer side as well as the consumer side in terms of syntax and mechanics.

## Unresolved questions

How to handle generics? Imagine something like `ember-power-select`:

```ts
interface PowerSelectBlocks<T> {
  default: [item: T];
}

export default class PowerSelectComponent<T> extends Component<..., PowerSelectBlocks<T>> {}
```

How is that information available to tools like the language server, so they can provide accurate suggestions?
