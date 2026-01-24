---
stage: accepted
start-date: 2025-12-12 # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - framework
  - learning
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1158
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

# Extend Component Signature for Styling

## Summary

Establish a place to document how to style your components in a structured
format.

## Motivation

CSS is developing at rapid speed an brought us developers a ton of useful
features to design our components. There is multiple levels at which we can
customize the styling for a component from the outside. Yet we are missing a
point to document these customizations aka CSS API.

Most important points of customization are (explained in detail below):

- CSS custom properties
- Parts
- Provided custom properties

The aim of this RFC is to close this gap and provide a format that can be picked
up by automated tooling to generate docs from it.

A good example is [Shoelace](https://shoelace.style) on how
they document their styling customizations. For example their
[Avatar](https://shoelace.style/components/avatar) component lists [custom
properties](https://shoelace.style/components/avatar#custom-properties) and
[parts](https://shoelace.style/components/avatar#parts).

I talked about a Styling signature in my 2022 talk about [Component
APIs](https://youtu.be/AQ3Jwb9vGqM?t=1824) for more detailed insight here.

## Detailed design

Extend the existing [component signature](./0748-glimmer-component-signature.md)
with a structured format to document styling customizations aka CSS API.
Extending component signature also means, this has no affect on the runtime.

Here is an exemplary signature for the Shoelace `Avatar` component:

```ts
interface AvatarSignature {
  Style: {
    CustomProperties: {
      '--size': 'The size of the avatar.'
    };
    Parts: {
      'base': 'The component’s base wrapper.';
      'icon': 'The container that wraps the avatar’s icon.';
      'initials': 'The container that wraps the avatar’s initials.';
      'image': 'The avatar image. Only shown when the `image` attribute is set.'
    }
  }
}
```

> [!NOTE]
> I use `Style` as name here, other candidates are `CSS` or `Styling`.

Using an object to document styling, a safe namespace is established as well as
having the option for future extension.

### `CustomProperties`

CSS custom properties are styling _inputs_ into a component, similar to `Args`.
They are the customization options with the lowest barrier and
can drive quite an impact. The concept is based on the [3+1 strategies idea by Lea
Verou](https://lea.verou.me/blog/2021/10/custom-properties-with-defaults/)
where she explains public and private CSS custom properties to provide a
stable API.

To document CSS custom properties, an object is sufficient enough with the key
as custom property and the value as explanation.

```ts
interface AvatarSignature {
  Style: {
    CustomProperties: {
      '--size': 'The size of the avatar.'
    };
  }
}
```

### `Parts`

Using [`part`](https://developer.mozilla.org/en-US/docs/Web/API/Element/part)
allow subelements of a component to be addressed and therefore styled.
Originally with
[`::part()`](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Selectors/::part)
selector as a way into shadow-dom, but also can be addressed with classic
`[part="some-name"]` attribute selector.

When `CustomProperties` are somewhat similar to `Args`, then `Parts` is somewhat
similar to `Blocks`. Like `CustomProperties` an object is sufficient to document
their usage with the key as parts name and the value as explanation.

```ts
interface AvatarSignature {
  Style: {
    Parts: {
      'base': 'The component’s base wrapper.';
      'icon': 'The container that wraps the avatar’s icon.';
      'initials': 'The container that wraps the avatar’s initials.';
      'image': 'The avatar image. Only shown when the `image` attribute is set.'
    }
  }
}
```

The current API for this is to provide specific `Args`, such as `@initialsClass`
or `@imageArgs` as a workaround for a propor solution to this problem.

### `Provides` (Questionable)

At times components do use certain CSS custom properties and their presence has
an impact onto the descendents of that component. When CSS custom properties are
the _inputs_ of a component, you can think of `Provides` as the _output_ for a
component.

Let's say you have a [`flow`](https://piccalil.li/blog/flow-utility/) and your
component is using that and also defining a value for `--flow-space`, a
descended of your component might also be using the `flow` utility and set
`--flow-space` accordingly or else you likely face design shifts.

> [!WARNING]
> This is very much debateable to have it in, in the first place. There is some
> usefulness in this, but also requires a lot of effort and willingness from
> engineers to document it and keep it updated, too.

### Tooling Support

The idea for a structured format is to include tooling to autogenerate
documentation from it.

#### Automated Documentation

Since the format is very well machine readable, tools such as
[`typedoc`](https://typedoc.org/) or
[`@microsoft/api-documenter`](https://www.npmjs.com/package/@microsoft/api-documenter)
can process this information.
In a second effort API UI tools can display these information, such as
[Storybook](https://storybook.js.org/) or
[kolay](https://github.com/universal-ember/kolay).

#### Validation / Verification

CSS (in contrast to Typescript) has no compiler that can check for the validity
of documented styling (in contrast to `Element`, `Args` or `Blocks` when using them = autocomplete in your IDE)

It provides a chance for tooling to emerge, eg. run a validator on top of
`ember-scoped-css` as potentially all data is available and can be parsed.

## How we teach this

Add it to the docs and let people know about it.

## Drawbacks

There are no drawbacks by the design itself. This can be seen as an addon for
people who will find this useful. It comes at no cost for the framework itself.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
