---
Stage:
Start Date: 2021-05-13
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/748
---

## Summary

In TypeScript, the `@glimmer/component` base class currently has a single `Args` type parameter. This parameter declares the names and types of the arguments the component expects to receive.

This RFC proposes a change to that type parameter to become `Signature`, capturing more complete information about how components can be used in a template, including their expected **arguments**, the **block parameters** they provide, and what type of **element(s)** they apply any received attributes and modifiers to.

This RFC is based in large part on prior work by [@gossi] and on learnings from [Glint].

[@gossi]: https://github.com/gossi
[Glint]: https://github.com/typed-ember/glint

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

Given a component with this template:

```hbs
<div class="notice notice-{{@kind}}" ...attributes>
  {{yield this.dismiss}}
</div>
```

This RFC proposes that a TypeScript backing class currently written like this:

```ts
import Component from '@glimmer/component';

export interface NoticeArgs {
  /** The kind of message displayed in this notice. Defaults to `info`. */
  kind: 'error' | 'warning' | 'info' | 'success';
}

export default class Notice extends Component<NoticeArgs> {
  // ...
}
```

Would instead, if fully typed, be written like this:

```ts
import Component from '@glimmer/component';

export interface GreetingSignature {
  Element: HTMLDivElement;
  Args: {
    /** The kind of message displayed in this notice. Defaults to `info`. */
    kind: 'error' | 'warning' | 'info' | 'success';
  };
  BlockParams: {
    /**
     * Any default block content will be displayed in the body of the notice.
     * This block receives a single `dismiss` parameter, which is an action
     * that can be used to dismiss the notice.
     */
    default: [dismiss: () => void];
  };
}

export default class Greeting extends Component<GreetingSignature> {
  // ...
}
```

By nesting the existing `Args` type under a top-level _signature_ type, we create a place to provide additional type information for other aspects of a component's behavior. One key element of this design is that eases future evolution of the signature, as including additional keys with new meaning won't be a breaking change.

In addition to `Args`, two other signature members are proposed here: `BlockParams` and `Element`. A detailed explanation of each of these follows.

### `Args`

The `Args` signature member works exactly as the top-level `Args` type parameter does in `@glimmer/component@1.x` prior to this RFC. That is, it determines the type of `this.args` in the component backing class, as well as acting as an anchor for human-readable documentation for specific arguments.

A component that accepts no arguments can omit the `Args` key from its signature.

### `BlockParams`

The `BlockParams` member dictates what blocks a component accepts and specifies what parameters, if any, it provides to those blocks.

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

The `BlockParams` type maps the block names a component accepts to a tuple type representing the parameters those blocks will receive. A component that never yields to any blocks may omit the `BlockParams` key from its signature.

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
  BlockParams: {
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
  BlockParams: {
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

### `Element`

The `Element` member of the signature declares what type of DOM element(s), if any, this component may be treated as encapsulating. That is, setting a non-`null` type for this member declares that this component may have HTML attributes applied to it, and that type reflects the type of DOM `Element` object any modifiers applied to the component will receive when they execute. Omitting `Element` or explicitly declaring it as `null` indicates that a component does not accept HTML attributes and modifiers at all.

For example, [`{{animated-if}}`] would omit `Element` from its signature, as it emits no DOM content. Even if you invoked it with angle-bracket syntax, any attributes or modifiers you applied wouldn't go anywhere.

On the other hand, [`<ResponsiveImage>`] would set `Element: HTMLImageElement`, as the element in its template that it ultimately spreads `...attributes` on to is an `<img>`.

While it's not common, occasionally components might forward `...attributes` to different types of elements in their template:

```hbs
{{#if @destination}}
  <a ...attributes href={{@destination}}>{{yield}}</a>
{{else}}
  <span ...attributes>{{yield}}</span>
{{/if}}
```

For such cases, components can use a union type for their `Element`. In the case of the template above, the signature would have `Element: HTMLAnchorElement | HTMLSpanElement`.

[`{{animated-if}}`]: https://ember-animation.github.io/ember-animated/docs/api/components/animated-if
[`<ResponsiveImage>`]: https://github.com/kaliber5/ember-responsive-image#the-responsiveimage-component

## How we teach this

### Documentation

TypeScript is [not (currently) officially supported by Ember](https://github.com/emberjs/rfcs/pull/724), and as such none of the framework guides or API documentation mention the existing `Args` type parameter today. The upshot of this is that, while the `Args` type parameter is well known by the portion of the Ember community that works with TypeScript, there's no official documentation to be updated.

The [`ember-cli-typescript` website], however, does have a section on working with components in TypeScript that deals almost exclusively with the `Args` parameter and `this.args` on the backing class. We should update and expand this documentation to cover the other concepts included in the `Signature` type.

[`ember-cli-typescript` website]: https://docs.ember-cli-typescript.com/ember/components#understanding-args

### Naming

The `Signature` concept itself has been [used in Glint] and broadly well received and understood, but the naming of each of the three member elements has received some discussion. While the names proposed above are what we currently believe to be best suited from both pedagogical and API-consistency perspectives, we're certainly interested in community feedback at this stage.

[used in Glint]: https://github.com/typed-ember/glint#component-signatures

#### `Args`

Early discussions have largely been on board with `Args` as it stands, though the non-abbreviated option `Arguments` has been suggested as an alternative. Since `Args` aligns better with `this.args` on the backing class (as well as the way in which people often colloquially discuss `@arg` values), we're continuing to propose `Args` here.

#### `Element`

Among early adopters of Glint there has been some confusion as to what the purpose of this key is. Generally "it's the concrete place your `...attributes` ultimately land after being passed down through components" has worked as an explainer, but the fact that an explainer is needed may indicate that `Element` isn't a clear name on its own.

That said, even among those who were initially unclear what the purpose of `Element` was, no one has been able to come up with an alternative proposal. Attempts at more explicit formulations like `UltimateSplattributesTarget` don't quite roll off the tongue 🙂

#### `BlockParams`

This key has easily been the largest topic of discussion among the three `Signature` members. Currently Glint uses `Yields` for this concept, but developers have given consistent feedback that that doesn't fit with `Args` and `Element` in their mental model.

`Blocks` has been the most commonly-suggested alternative, but that doesn't quite align with the information being captured. Blocks are the chunks of content passed _in_ to a component by the author invoking it, while this signature member captures what parameters (if any) the component will pass _out_ to those blocks.

For all three signature members proposed here, Glint will align its API with whatever terminology this RFC lands on.

### Migration

We can implement this change in `@glimmer/component` 1.x in a backwards compatible way, allowing for a deprecation period before moving exclusively to the signature approach in 2.0. Because capitalized arg names are currently illegal, any valid signature will not represent valid arg names, so the backing class can [accept both formats](https://tsplay.dev/Nr2rzN) and distinguish which was intended.

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
class Component<Args = {}, BlockParams = {}, Element = null> {
  // ...
}
```

This has the advantage of not requiring current users of the `Args` parameter to change their code, but suffers many of the same ergonomic issues as regular functions do when they begin to accrue many positional parameters. Authors need to remember which parameters appear in which order when using the `Component` type, and they need to be aware of the appropriate default values for earlier parameters to fill in when they only want to specify a later one.


### Only `Args`

One of the drawbacks mentioned above is that the `Element` and `BlockParams` signature members described in this RFC are functionally inert in a vanilla TypeScript project. This leaves them with about the same status as comments: potentially helpful when left by a well-meaning author, but without any checks to ensure that they're accurate and that they stay up-to-date as the implementation changes.

An alternative would be to still introduce the `Signature` type in `@glimmer/component` but _only_ formalize the `Args` member, leaving other tooling to define the semantics of any additional signature members they might be interested in. While this would simplify the overall proposal somewhat, what a component `{{yield}}`s and what it does with its `...attributes` are core enough to a component's public interface that we believe they should be considered first-class rather than having individual tools reinvent them, potentially in mutually-incompatible ways.
