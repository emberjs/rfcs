---
Stage: Proposed
Start Date: 2021-05-13
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/748
---

## Summary <!-- omit in toc -->

In TypeScript, the `@glimmer/component` base class currently has a single `Args` type parameter. This parameter declares the names and types of the arguments the component expects to receive.

This RFC proposes a change to that type parameter to become `Signature`, capturing more complete information about how components can be used in a template, including their expected **arguments**, the **blocks** they provide, and what type of **element(s)** they apply any received attributes and modifiers to.

This RFC is based in large part on prior work by [@gossi] and on learnings from [Glint].

[@gossi]: https://github.com/gossi
[Glint]: https://github.com/typed-ember/glint


### Outline <!-- omit in toc -->

- [Motivation](#motivation)
- [Detailed design](#detailed-design)
  - [`InvokableComponentSignature`](#invokablecomponentsignature)
  - [`GlimmerComponentSignature`](#glimmercomponentsignature)
  - [Updated type for Glimmer `Component`](#updated-type-for-glimmer-component)
  - [Example](#example)
  - [`Args`](#args)
  - [`Blocks`](#blocks)
  - [`Element`](#element)
    - [Components With Multiple or Varying Elements](#components-with-multiple-or-varying-elements)
- [How we teach this](#how-we-teach-this)
  - [Documentation](#documentation)
  - [Migration](#migration)
    - [Codemods](#codemods)
    - [Deprecation Warnings/Linting](#deprecation-warningslinting)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [Additional Positional Type Parameters](#additional-positional-type-parameters)
  - [Only `Args`](#only-args)
  - [Naming](#naming)
    - [`Args`](#args-1)
    - [`Element`](#element-1)
    - [`Blocks`](#blocks-1)

## Motivation

When a developer goes to invoke a component in a template, there are a variety of questions about its interface they need to be able to answer in order to determine how to use it correctly.

1. What arguments does this component accept?
   - Which arguments are required and which are optional?
   - What does each argument do?
   - What types of values am I expected to pass for a given argument?
2. How can I nest content within this component?
   - Does it accept a default block? Named blocks?
   - In what context will my block content appear in the component's output?
   - For any blocks I write, what parameters does the component expose to them?
3. In what ways can I treat this component like a regular HTML/SVG element?
   - Can I pass attributes to it?
   - If I apply a modifier, what kind(s) of DOM `Element` object will the modifier see?

In the Glimmer `Component` base class today, a TypeScript component author can answer the first bucket of questions above by defining an `Args` type for the component class. Using this structured documentation, developer tooling can surface information about a component's arguments to consumers without requiring them to go read either ad-hoc prose or the implementation itself to determine the component's intended usage.

For the second and third sets of questions, however, there is no such affordance for providing structured answers.

The goal of this change is to enable tooling to provide a more complete picture for developers of the ways in which Glimmer components can be used. The word "tooling" here is used in broad terms, including:

 - API/design system documentation generators, like [Storybook] or [Hokulea]
 - Language servers that provide hover information, go-to-definition, etc. like [TypeScript] or [uELS]
 - Template-aware static analysis and typechecking systems, like [Glint] or [`els-addon-typed-templates`]

[Storybook]: https://storybook.js.org/
[Hokulea]: https://github.com/hokulea/hokulea
[TypeScript]: https://www.typescriptlang.org/
[uELS]: https://github.com/lifeart/ember-language-server
[Glint]: https://github.com/typed-ember/glint
[`els-addon-typed-templates`]: https://github.com/lifeart/els-addon-typed-templates


## Detailed design

This RFC proposes two new TypeScript APIs:

1. A fully general `InvokableComponentSignature` type which can capture *all* the relevant details of components in the Glimmer VM, with
2. A user-facing Glimmer Component `Signature` type that enables authors to answer the questions outlined above about their Glimmer components in a more concise and convenient way.

Throughout, these new interfaces intentionally use `PascalCase` for names representing *types*, including types nested in interfaces:

- to match the general TypeScript ecosystem norm that types are named in `PascalCase`
- to thereby distinguish clearly between type- and value- names in contexts which might otherwise be ambiguous
- ***to enable adopting this in a backwards-compatible way with the existing Glimmer Component type definition***

The third of these is the most critical, and much more strongly motivates the others.

These interfaces are *not* importable, because importing them does not give any value to consumers: use of `extends` does not add any constraints (because of the optional-ity discussed below).


### `InvokableComponentSignature`

The base `InvokableComponentSignature` type is an intentionally-verbose form, which end users will not *normally* write, but it is legal to do so.

Two points to notice about the signature:

1. All of the fields for what users author are optional. They are resolved into the appropriate *non*-optional representations by type machinery in Glint. For example, in a Glimmer component the `args` is never undefined; it is minimally an empty object.
2. This provides the foundation for further extensions of each of these as needed. For example, if we were to add support for named block params (in addition to today’s support for positional arguments), they could be added as a `Named` field in the interface, just like the `Positional` field present today.

```ts
// A base signature type which represents items which can be invoked with
// arguments, and could be reused for helpers, modifiers, etc.
interface InvokableSignature {
  Args?: {
    Named?: Record<string, unknown>;
    Positional?: unknown[];
  };
}

interface InvokableComponentSignature extends InvokableSignature {
  // The `null` means "this does not use ...attributes"
  Element?: Element | null;
  Blocks?: {
    [blockName: string]: {
      Positional?: unknown[];
    }
  }
}

interface InvokableGlimmerComponentSignature extends InvokableComponentSignature {
  Args?: {
    Named?: Record<string, unknown>;
    // empty tuple here means it does not allow *any* positional params
    Positional?: [];
  }
}
```

As suggested by this lattermost form, specific component implementations may also represent a *subset* of the full signature. Future component implementations may take advantage of this just as future extensions to the system may take advantage of the ability to add more information into the signature.


### `GlimmerComponentSignature`

Since the fully-expanded form is quite verbose, we also provide a much smaller interface users can write, which we expand “under the hood” into the full signature as well as onto the class body for `args`. Users are *allowed* to supply the fully-expanded form; they just do not *have* to.

```ts
interface GlimmerComponentSignature {
  Args?: {
    [argName: string]: unknown;
  };
  Blocks?: {
    [blockName: string]: unknown[]
  }
  Element?: Element | null;
}
```

As with the `InvokableComponentSignature`, the type that authors *must* write is simply empty: all the fields are optional. Once the type is attached to the component, it is resolved to the appropriate default value. For example, a component’s `args` property is *always* an object, but it may be empty. If `Args` is not supplied, it is defaulted the empty object. We cover the details of this defaulting behavior for each field below.


### Updated type for Glimmer `Component`

With these signature types defined, we can update the type for the Glimmer `Component` class in a backwards-compatible way:

- adding support for the new expanded type signature
- continuing to support all *current* uses of the type signature
- dropping the `extends` constraint, which provides little-to-no actual constraining value

Previously:

```ts
class Component<Args extends {} = {}> {
  readonly args: Args;
}
```

Updated:

```ts
class Component<Signature> {
  readonly args: ComponentArgs<Signature>;
}
```

Here, the `ComponentArgs` type will be a type utility which can resolve the named arguments to the component (the exact mechanics are an implementation detail, but you can see one possible design in [this playground][fully-working]). Users may pass any of:

- an arguments-only definition, as has been recommended up till now in the Glimmer Component v1 era
- the `GlimmerComponentSignature` short-hand form
- the expanded `InvokableGlimmerComponentSignature` form
- the fully-expanded `InvokableComponentSignature` form

[fully-working]: https://www.typescriptlang.org/play?#code/JYWwDg9gTgLgBAbzgUwB5mQYxgFQJ4YDyAZnAL5zFQQhwDkaG2AtDAcnQNwBQ3AJlgA2AQyjI4mCADsAzvACi4NgC44AVynAAjmvEy8IAEYRBPNhjiKwbQoYBWWeAF5EcANpW2AXQD8qmFC65Dzc5uIA4sgwhFDygjLIADy2dgA0cADS6QBiwoKChsKYANYAfHAuGSioMMhSfDJwxch4EKQpcD5wKW4ZXnCqufmFJSFhcACCUADmMtnQiQDK5S7NraSL1bX1jQBEU7O7cAA+cLtxyCB1MEenuwBCghAlMkcA9G9wAJKNMAAWwEawjgMmA0ykwhgajEPm4cE6ZwOry2dQaTRabTgmw+cAAIhBkL8AY1QeDIdDxH9hAA3cQAAyRdLgAmIwCkyD4sPh8K6izc+xmr36aG2aKQADlhFdOaoIPZHJw4AAFCCgmDAaR5PzqKTFKQQADuUjc-QoOKm4n+LTgTyk0yp9TgeTEwj4eC53J5rkl0tUkWisXiST5AsOXnSdB9HLo6U8eBSjlKipVao1EMEfqiMTiCSW-KRu3D9BTwHVmsEMfcXnKZDhnoG3qlHNUIYLXmTqtLabyqhN5Dr8NUEqbfFUcYT2A7qfLvdNdaHcCjo6xU67M6rwV4bwAVNu4OFgLSpE6pHgnflDRyQX9oP9hI7iNBaJjgaSIVCxOkBDI1NNRI1SzgGAIDgUtGmINR8mqMB7wEPggPYOBtzeUJEPkdBYMWMF3wpRIcBWRA6wuK4pBgVRzkES5riOEVUUaNZMRwBEcH5YjqP6VQpEg0w6yRVQkXmKA8KTOtHmeYoZHIsSXhomo6PRdY4BwAcEQQFTPTcaTilA48GNIFiHieGSvC8VQDK0oVNKM4phTknYdT1Q1jS8dT6y6JASzLdMzP5CzCys8S53resfMM8TLK09tXNrT0xyUeN5UnbgyBCHEAAFpkEUArigN5JHAaRrjgABGAA6AAGbg2VqKBiCKcRCEEPhFkwG8TCRQj4UfCBVDkKA2WmHha24HFdmENR-mgAbKCfSEjghaVmFEWZmGkQQ8Cq0jkFq+qsRvWAHT4JFCCkdbOsmQUFxU7reoCAaeHhWthpxWkoGMBJ0kMCabWQGlpogb7GCyzBAM87tBE2mq6swcQl2O06zzU+E+PO+ElyHVybpBO67Qe7kYv7Z7Ple97xAWjklsFVaEbgAAKOl0Jg+osLJD9kCZGQb0g+DDHEYF9QgMA4GkIDiQASkh7bofEbJuPhs6kYu2YMc9MH1xNPG0ZHFX6yxvr7pUgmUqIyiSLIuAuPyPGLLi6wEocJLhsYRx8CIYhEkZzDsPJMRkialq2sEJFSlKWmxdK4D5B0PJXeQEhEll-J5bwUOxZ4Z3sFj+PPeZ722aWfa73qZOQ7DiOICjtQY-YePE6DwUTvW1P0-QF2a-dnPmrz3C4YbhHS-DyPo8ELP3brkuw5bphcHbj2MNz1ncPHvum9T8vK+rt2E7lleU8nrdPjGib9umx8oBAObmWQOrIPgQxrOFhHJZ2mG9tvQ6LMbxHROsyTUavm+ghzZuCuDIGQwhpjIFuv1O0UVHrJQPnAEmqpxAsnGkAuA99xKPzOvTTuLMcJiA5lzJqmC+YWwgILR+otAQS2qlLXaddP400VmxUinFuJ4xRsOX0lh4oThgKuLyPYqyKmNvCG2-80G3x1qrTswiMzuD1jjaYcD8Z1mNk7VumdZ74O7r7RYhcP6-y-gPdew9R7b3yMwpu+8M4zy3noxevsmEmP7mvIeVcR6z1ceFUx+8RqH3GpNGB0wZrn0vsgU2RU1obXoS-cQhj36wTYdEFhJsqLsLgAACRwAAWQADIAGF7zUmEDIVJQ1EHIISCgaJpEcFnjwfPLuzj2bXgBqQ3mToKFUJFv8Whz9pZwDrqkr+51JHjkSoI3il1Gy8KmQ7QRyp5Hg1nJueEqTVC5MKSUqQZSKn1JmVo6elinGEODEYlJRzTEeIrhYnx3Exn9zsdohxccO4tIIT7JIoybnuLLp4zeHyrGCGebYtOiCIL5DPJAMAkFIQcnSNCs6jBYJXjfD8umDMvn6PaZzTpPNyECyFv08WQzdryFengAZdp7gQKiedbhKl0auGUaEjZ3I1beXcKWS4HCjDbTUeIywRztn5OKaU8plSf7hVkQA9B5tFZyOnDytwZTBC6F6gYYwggXKeiNgg7g9izm4raR7altLpj0sgYIMxQLvGOMtQCOlDK7UBPGEUmgkB2SkSREsAiAkFjLDcJGEcdAor8CEKIcQmARBgLgF6gqvqYABvOi6PgsSnRzKDUJENYbpQRqNXG8pjRGrNTYJRFE9kk0+uuH7ZqrVKH11mOURWkD4DdTDv-MQH5jwDJkKVZag7up4yerwEtCbxTIANIsSt4haI1u9YVUiBdknF13m2usHaZoQG7cquAvboT9uJEOwUpVR0aOSkAA


### Example

Given a component with this template:

```hbs
<div class="notice notice-{{@kind}}" ...attributes>
  {{yield this.dismiss}}
</div>
```

A TypeScript backing class currently written like this:

```ts
import Component from '@glimmer/component';

export interface NoticeArgs {
  /** The kind of message displayed in this notice. Defaults to `info`. */
  kind?: 'error' | 'warning' | 'info' | 'success';
}

export default class Notice extends Component<NoticeArgs> {
  // ...
}
```

Would instead, if fully typed, be written like this:

```ts
import Component from '@glimmer/component';

export interface NoticeSignature {
  Element: HTMLDivElement;
  Args: {
    /** The kind of message displayed in this notice. Defaults to `info`. */
    kind?: 'error' | 'warning' | 'info' | 'success';
  };
  /**
   * Any default block content will be displayed in the body of the notice.
   * This block receives a single `dismiss` parameter, which is an action
   * that can be used to dismiss the notice.
   */
  Blocks: {
    default: [dismiss: () => void];
  };
}

export default class Notice extends Component<NoticeSignature> {
  // ...
}
```

By nesting the existing `Args` type under a top-level _signature_ type, we create a place to provide additional type information for other aspects of a component's behavior. This, along with the fully-expanded/desugared form, is the key element of this design which enables future evolution of the signature, as including additional keys with new meaning won't be a breaking change.


### `Args`

The `Args` signature member works exactly as the top-level `Args` type parameter does in `@glimmer/component@1.x` prior to this RFC. That is, it determines the type of `this.args` in the component backing class, as well as acting as an anchor for human-readable documentation for specific arguments.

**A signature with no `Args` indicates that its component does not accept any arguments. The type of the `args` field on a Glimmer component class is an empty object.**


### `Blocks`

The `Blocks` member dictates what blocks a component accepts and specifies what parameters, if any, it provides to those blocks. All blocks must named explicitly.[^blocks-sugar] When the component only yields to the default block, simply name the `default` block:

```ts
Blocks: {
  default: [name: string];
}
```

```hbs
{{yield "Tomster"}}
```

When there are multiple blocks, each is named to indicate what parameters a component's named blocks provide:

```ts
Blocks: {
  header: [];
  body: [item: T; index: number]
}
```

```hbs
{{yield to="header"}}

{{#each items as |item index|}}
  {{yield item index to="body"}}
{{/each}}
```

This means that if a component also accepts both a default block _and_ other named blocks, it can specify the default block by name (`default: [...]`) in its `Blocks` in exactly the same way a consumer of the component might use `<:default>` to pass a default block alongside named ones when invoking a component.

**A signature with no `Blocks` indicates that its component never yields to any blocks.**

The [yieldable named blocks RFC] and recent versions of the [Component guides] discuss blocks in some depth, but since named blocks in particular are relatively new to the community, brief definitions based on those in [RFC 678] are included here for clarity.

[yieldable named blocks RFC]: https://github.com/emberjs/rfcs/blob/master/text/0460-yieldable-named-blocks.md
[Component guides]: https://guides.emberjs.com/release/components/block-content/
[RFC 678]: https://github.com/emberjs/rfcs/pull/678

<details>
  <summary>Blocks, Block Parameters and Yielding</summary>

  - **Block**

    A block is a section of content that a template author provides to a component when invoking it. Many components accept a _default_ block, and components invoked using curly braces may also accept an _else_ block.

    ```hbs
    <Modal>
      This is a default block.
    </Modal>

    {{#if-a-coin-flip-is-heads}}
      This is also a default block.
    {{else}}
      This is an else block.
    {{/if-a-coin-flip-is-heads}}
    ```

    For angle-bracket components, the above example with an _implicit_ default block could also be written with an _explicit_ one:

    ```hbs
    <Modal>
      <:default>This is another default block.</:default>
    </Modal>
    ```

    This syntax, using `<:identifier>` to delimit block contents, allows authors to provide one or more _named blocks_ to a component:

    ```hbs
    <Modal>
      <:header>
        This is the header block.
      </:header>
      <:body>
        This is the body block.
      </:body>
    </Modal>
    ```

  - **Block Parameters**

    Blocks may also receive _parameters_ from the component that they're provided to, using `as |identifier ...|` syntax.

    For an implicit default block, the parameters are exposed from the top level component:

    ```hbs
    <Modal as |close|>
      <button {{on "click" close}}>Close</button>
    </Modal>
    ```

    For named blocks, different parameters may be exposed individually to each block:

    ```hbs
    <Modal>
      <:header>Close me</:header>
      <:body as |close|>
        <button {{on "click" close}}>Click!</button>
      </:body>
    </Modal>
    ```

  - **Yield**

    Yielding is how a component invokes a provided block, optionally exposing block parameters. An example from [RFC 460](https://emberjs.github.io/rfcs/0460-yieldable-named-blocks.html#block-parameters):

     ```hbs
     <article>
       <header>{{yield @article.title to='header'}}</header>
       <section>{{yield @article.body to='body'}}</section>
     </article>
     ```
</details>

The `Blocks` type maps the block names a component accepts to a tuple type representing the parameters those blocks will receive.

As a concrete example, the `BlogPost` component in the [Block Parameters section] of the Ember guides looks like this:

[Block Parameters section]: https://guides.emberjs.com/release/components/block-content/#toc_block-parameters

```hbs
{{yield @post.title @post.author @post.body}}
```

```hbs
<!-- usage -->
<BlogPost @post={{@blogPost}} as |postTitle postAuthor postBody|>
  <img alt="" role="presentation" src="./blog-logo.png">

  {{postTitle}}

  {{postBody}}

  <AuthorBio @author={{postAuthor}} />
</BlogPost>
```

The signature for this component might look like:

```ts
export interface BlogPostSignature {
  Args: { post: Post };
  Blocks: {
    default: [postTitle: string, postAuthor: string, postBody: string];
  };
}
```

Block parameters may also depend on the types of args a component receives. For example, for a fancy table component:

```hbs
<FancyTable @items={{this.people}}>
  <:header>
    <td>Name</td>
    <td>Age</td>
  </:header>

  <:row as |person|>
    <td>{{person.name}}</td>
    <td>{{person.age}}</td>
  </:row>
</FancyTable>
```

The backing class and signature might look something like:

```ts
export interface FancyTableSignature<T> {
  Args: {
    /** The items to be displayed in the fancy table, each corresponding to one row. */
    items: Array<T>
  };
  Blocks: {
    /** Any header contents for the table, broken into cells. */
    header: [];

    /** Content to be rendered for each row, receiving the corresponding item and its index. */
    row: [item: T, index: number];
  };
}

export default class FancyTable<T> extends Component<FancyTableSignature<T>> {
  // ...
}
```

While the runtime design of named blocks currently only permits components to yield parameters out to them, community members have suggested possible ways of making blocks more akin to components themselves, potentially accepting `@args` or attributes, or even themselves accepting further nested blocks. Should such a design ever become reality, the shape of a signature might evolve to become more recursive. Today, however, there are many open questions about how such functionality would actually work for authors, and so we keep the proposed format here simple.

[^blocks-sugar]: We recognize that it might be desirable to have a shorthand for the very common case of having a single default block. However, none of the designs we have seen so far are satisfactory across the board in terms of teaching and mental model, so we are using this expanded form which *does* satisfy those constraints. Over time, we hope to come up with a nice bit of “sugar” that works there, but we are not *blocked* on finding that sugar.


### `Element`

The `Element` member of the signature declares what type of DOM element(s), if any, this component may be treated as encapsulating. That is, setting a non-`null` type for this member declares that this component may have HTML attributes applied to it, and that type reflects the type of DOM `Element` object any modifiers applied to the component will receive when they execute.

**A signature with no `Element` or with `Element: null` indicates that its component does not accept HTML attributes and modifiers at all.** While applying attributes or modifiers to such a component wouldn't produce a runtime error, it still likely constitutes a mistake on the author's part, similar to [passing an unknown key in an object literal][ecp].

For example, [`{{animated-if}}`] would omit `Element` from its signature, as it emits no DOM content. Even if you invoked it with angle-bracket syntax, any attributes or modifiers you applied wouldn't go anywhere.

On the other hand, [`<ResponsiveImage>`] would set `Element: HTMLImageElement`, as the element in its template that it ultimately spreads `...attributes` on to is an `<img>`.

[`{{animated-if}}`]: https://ember-animation.github.io/ember-animated/docs/api/components/animated-if
[`<ResponsiveImage>`]: https://github.com/kaliber5/ember-responsive-image#the-responsiveimage-component

The `Element` member is of particular relevance for the modifiers that consumers can apply to a component. In a system using this information to provide typechecking, any modifiers applied to its component must be declared to accept the component's `Element` type (or a broader type) as its first parameter, or else produce a type error.

- A component with `Element: Element` can only be used with modifiers that accept _any_ DOM element. Many existing modifiers in the ecosystem, such as `{{on}}` and everything in `ember-render-modifiers`, fall into this bucket.

- A component with e.g. `Element: HTMLCanvasElement`, may be used with any general-purpose modifiers as described above _as well as_ any modifiers that specifically expect to be attached to a `<canvas>`.

- A component whose `Element` type is a [union of multiple possible elements](#components-with-multiple-or-varying-elements) can only be used with a modifier that is declared to accept _all_ of those element types. This behavior is, in fact, the point—modifiers are essentially callbacks that receive the element they're attached to, and so the [normal considerations][variance] for typing callback parameters apply.

[ecp]: https://www.typescriptlang.org/docs/handbook/interfaces.html#excess-property-checks
[variance]: https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)

#### Components With Multiple or Varying Elements

While it's not common, occasionally components might forward `...attributes` to different types of elements in their template:

```hbs
{{#if @destination}}
  <a ...attributes href={{@destination}}>{{yield}}</a>
{{else}}
  <span ...attributes>{{yield}}</span>
{{/if}}
```

For such cases, components can use a union type for their `Element`. In the case of the template above, the signature would have `Element: HTMLAnchorElement | HTMLSpanElement`. Correspondingly, any modifiers used with such components would need to accept any of the possible types of DOM elements declared.

Similarly, a component that may use `...attributes` on an `<a>` element or may not spread them at all might write: `Element: HTMLAnchorElement | null`. In such cases, ecosystem tooling consuming this type information should treat it as legal to use any modifiers that accept an `HTMLAnchorElement`, since they wouldn't ever be invoked for the `null` scenario.

In cases where the distinction between possible elements is key to the functionality of the component and can be statically known based on the arguments passed in, the component author may choose to capture this as part of the signature at the expense of additional type-level bookkeeping.

<details><summary>Gritty details</summary>

The particular shape/value of arguments is something that varies from instance to instance of the component, and the standard tool in TypeScript for handling these cases is to introduce a _type parameter_ on the type(s) in question.

For the `{{#if @destination}}` example above, the implementation might look like this:

```ts
interface MaybeLinkSignature<Destination extends string | undefined> {
  Args: {
    destination: Destination;
    target?: string;
  };
  Blocks: {
    default: []
  };
  Element: Destination extends string
    ? HTMLAnchorElement
    : HTMLSpanElement;
}

export default class MaybeLink<Destination extends string | undefined>
  extends Component<MaybeLinkSignature<Destination>> {
  // ...
}
```

This would allow consumers to use a modifier that requires an `HTMLAnchorElement` on a `MaybeLink` if and only if the `@destination` arg they pass is definitely a string.

Note, still, that general-purpose modifiers like `{{on}}` or `{{did-insert}}` would be usable with this component regardless of the type of `@destination`, or even if the author had simply typed `Element` as `HTMLAnchorElement | HTMLSpanElement` without the extra song-and-dance of explicitly capturing the `Destination` type.

Finally, template analysis tools can provide escape hatches in the same vein as TypeScript's `@ts-ignore` and `@ts-expect-error` for these or any cases where consumers have information that library authors haven't encoded in the type system.
</details>


## How we teach this

### Documentation

Ember is [in the midst](https://github.com/emberjs/rfcs/pull/724) of [adding official supported for TypeScript](https://github.com/emberjs/rfcs/pull/800), and as such none of the framework guides or API documentation mention the existing `Args` type parameter today. The upshot of this is that, while the `Args` type parameter is well known by the portion of the Ember community that works with TypeScript, there's no official documentation to be updated.

The [`ember-cli-typescript` website], however, does have a section on working with components in TypeScript that deals almost exclusively with the `Args` parameter and `this.args` on the backing class. We should update and expand this documentation to cover the other concepts included in the `Signature` type.

[`ember-cli-typescript` website]: https://docs.ember-cli-typescript.com/ember/components#understanding-args

When we add sections documenting the use of TypeScript to the official guides, add type information to the API docs, and and so on, we will migrate that discussion from the `ember-cli-typescript` documentation into the main site documentation.

Glint and its current documentation currently use earlier names and structures for the basic ideas in this signature, and will be updated to match this spec.


### Migration

As noted in [**Detailed Design**](#detailed-design), we can implement this change in `@glimmer/component` 1.x in a backwards compatible way, allowing for a deprecation period before moving exclusively to the signature approach in 2.0. Because capitalized arg names are currently illegal, any valid signature will not represent valid arg names, so the backing class can [accept both formats](https://tsplay.dev/Nr2rzN) and distinguish which was intended.

Ideally we'll be able to encourage authors to migrate their usage of `Args` in the same ways they're used to being nudged to move away from other deprecated APIs in the Ember ecosystem.


#### Codemods

A simple codemod to turn `Component<MyArgs>` into `Component<{ Args: MyArgs }>` would be straightforward, but it may also be feasible to migrate more completely by inferring information like block names and where `...splattributes` are used from colocated templates. The richness we pursue here will likely depend on the appetite of the community to explore what's possible.


#### Deprecation Warnings/Linting

While type-only deprecations aren't something the Ember ecosystem has dealt with much previously, it's a muscle we may want to begin building as [official TypeScript support] is under consideration.

The `@typescript-eslint` suite of packages supports writing [type-aware rules] in projects that provide appropriate configuration in their `eslintrc`, so one option available to use would be to use that functionality to allow users to lint for components that haven't yet been migrated to use a signature type. We could expose this as part of a standalone ESLint plugin, or perhaps make it available on an opt-in basis in `eslint-plugin-ember`.

[official TypeScript support]: https://github.com/emberjs/rfcs/pull/724
[type-aware rules]: https://github.com/typescript-eslint/typescript-eslint#can-we-write-rules-which-leverage-type-information


## Drawbacks

As with any change that deprecates supported behavior, there's an inherent cost associated with migrating the user base over to new patterns. One goal of this change, however, is to ease such migrations in the future: as the templating system evolves and new information becomes relevant to capture and document, the `Signature` type provides a place for that information to live without disrupting existing code.

The other potential drawback to this approach is that it introduces TypeScript type information that has no visible effect on the component using out-of-the-box tooling. Without a template-aware system like Glint or `els-addon-typed-templates` for validating components, nothing enforces that the signature declared is actually accurate. See the "Only `Args`" section under Alternatives below for further discussion of this point.


## Alternatives

### Additional Positional Type Parameters

Rather than wrapping the information we're interested in capturing in a `Signature` type, we could instead introduce further type parameters to the `Component` base class:

```ts
class Component<Args = {}, Blocks = {}, Element = null> {
  // ...
}
```

This has the advantage of not requiring current users of the `Args` parameter to change their code, but suffers many of the same ergonomic issues as regular functions do when they begin to accrue many positional parameters. Authors need to remember which parameters appear in which order when using the `Component` type, and they need to be aware of the appropriate default values for earlier parameters to fill in when they only want to specify a later one.


### Only `Args`

One of the drawbacks mentioned above is that the `Element` and `Blocks` signature members described in this RFC are functionally inert in a vanilla TypeScript project. This leaves them with about the same status as comments: potentially helpful when left by a well-meaning author, but without any checks to ensure that they're accurate and that they stay up-to-date as the implementation changes.

An alternative would be to still introduce the `Signature` type in `@glimmer/component` but _only_ formalize the `Args` member, leaving other tooling to define the semantics of any additional signature members they might be interested in. While this would simplify the overall proposal somewhat, what a component `{{yield}}`s and what it does with its `...attributes` are core enough to a component's public interface that we believe they should be considered first-class rather than having individual tools reinvent them, potentially in mutually-incompatible ways.


### Naming

The `Signature` concept itself has been [used in Glint] and broadly well received and understood, but the naming of each of the three member elements has received some discussion. While the names proposed above are what we currently believe to be best suited from both pedagogical and API-consistency perspectives, there are alternatives we could choose (and have explored).

[used in Glint]: https://github.com/typed-ember/glint#component-signatures


#### `Args`

Early discussions have largely been on board with `Args` as it stands, though the non-abbreviated option `Arguments` has been suggested as an alternative. Since `Args` aligns better with `this.args` on the backing class (as well as the way in which people often colloquially discuss `@arg` values), we're continuing to propose `Args` here.


#### `Element`

Among early adopters of Glint there has been some confusion as to what the purpose of this key is. Generally "it's the concrete place your `...attributes` ultimately land after being passed down through components" has worked as an explainer, but the fact that an explainer is needed may indicate that `Element` isn't a clear name on its own.

That said, even among those who were initially unclear what the purpose of `Element` was, no one has been able to come up with an alternative proposal. Attempts at more explicit formulations like `UltimateSplattributesTarget` don't quite roll off the tongue 🙂


#### `Blocks`

This key has easily been the largest topic of discussion among the three `Signature` members. Currently Glint uses `Yields` for this concept, but developers have given consistent feedback that that doesn't fit with `Args` and `Element` in their mental model. Moreover, the Framework team noted that it also doesn’t match with hoped-for iterations on the mental model after Polaris, where the notion of “yielding” (largely a holdover from Ember’s roots with many of its designers coming from the Ruby world) is less prominent or removed entirely in favor of new language/concepts.

An earlier draft also used `BlockParams`, but that was not readily extensible to capture future information about blocks. Since Blocks are the chunks of content passed _in_ to a component by the author invoking it, while this signature member captures what parameters (if any) the component will pass _out_ to those blocks, we also did not want to support a shorthand like `Blocks: []`, which could very easily be misread as “there are no blocks” instead of “there is a default block which yields no params”.
