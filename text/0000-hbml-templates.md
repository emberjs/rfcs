- Start Date: 2020-01-09
- Relevant Team(s): Ember.js
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# HBML Templates

## Summary

In this RFC, we propose to introduce support for a new kind of templates into
Ember.js – the Handlebars Markup Language, or HBML, with an `.hbml` extension
to distinguish them from regular `.hbs` tempaltes.

HBML templates are a variant of the current `.hbs` templates with the following
differences:

1. It specifically refers to the Ember's dialect of the Handlebars-in-HTML
   templating language, unlike regular `.hbs` template which generically refers
   to any applications of Handlebars, such as in Markdown or plain text files.

2. HBML templates always has [strict mode](./0496-handlebars-strict-mode.md)
   semantics, whereas `.hbs` which has "sloppy mode" semantics by default.

3. HBML templates supports template imports via a frontmatter syntax.

Today, a `.hbs` template in Ember may look like this:

```hbs
{{#if isNewUser}}
  <ProductTour @user={{user}} />
{{else}}
  <Notification as |dismiss|>
    Hello {{user.name}}, welcome back!
    <button {{on "click" dismiss}}><Icon @type="close" /></button>
  </Notification>
{{/if}}

{{outlet}}
```

The equivilant HBML template would look like this:

```hbs
---
import { on } from '@ember/modifier';
import Icon from 'ember-icon-pack';
import Notification from 'my-app/components/notification';
import ProductTour from 'my-app/components/product-tour';
---

{{#if this.isNewUser}}
  <ProductTour @user={{this.user}} />
{{else}}
  <Notification as |dismiss|>
    Hello {{this.user.name}}, welcome back!
    <button {{on "click" dismiss}}><Icon @type="close" /></button>
  </Notification>
{{/if}}

{{outlet}}
```

## Motivation

The introduction of [strict mode](./0496-handlebars-strict-mode.md) brought
with it two problems:

1. How do users opt-in to the strict mode semantics?

2. If there are no globals or dynamic resolutions, how do users access things
   like components and helpers in their templates?

The HBML proposal solves both of these problems at once, by using the new file
extension as an explicit opt-in to strict mode semantics, and by introducing
a template import syntax for brining items into scope.

A dedicated file extension for Ember templates also makes it easier to provide
good developer experience. As mentioned above, Handlebars is a general purpose
templating syntax designed to be embedded into many different context. While
`.hbs` files almost always refer to Handlebars-in-HTML templates in an Ember
app, this is not generally the case in the wider ecosystem.

As a result, when using the `.hbs` extension, editors and tools are not always
able to reliably provide the best language support (e.g. auto-completion and
syntax highlight of HTML tags) for these files. By declaring a dedicated file
extension for Ember's use case, these tools can implement more targeted support
for HTML and Ember-specific extensions to the Handlebars syntax. While it may
take some time for it to gain traction in the tooling ecosystem, we believe
this would be an improvement in the long run.

Finally, by adopting the JavaScript import syntax and its module semantics, it
also makes it much easier for tools to understand Ember templates. For example,
build tools can use the imports to build a dependency graph for the purpose of
tree-shaking, IDEs can more reliably provide features like "jump to definition"
and automated refactoring, TypeScript can use this information to locate type
definitions, etc. Of course, it also makes it easier for human readers to
follow where different variables come from.

## Detailed Design

### `.hbml` File Extension

* Editors
* GitHub, etc
* Compiler
* Build Tools
* Linters, Recast, etc
* TypeScript
* Inline `hbml` function
* `{{! use hbml }}` opt-in for `.hbs` files?

### Strict Mode Semantics

* See "Strict Mode" RFC
* No opt-out

### Import Declaration

* `---` must be beginning of file (no comments, whitespaces)
* Only JS import statements, JS whitespace and JS comments
* Optional newline after "closing" `---`, not part of content
* Merged with JS
* Renamed to avoid conflict with JS imports and preserve JS-side errors
* Special `import ... from '.';` to "import" from component js file
* Keywords/syntax does not need to be imported, cannot be passed around
* New import paths for built-ins?
* Any issues with other non-core "magic globals" that doesn't have import paths?
* How to use legacy non-co-located components? Does it matter?
* `hbml`: no explicit access to import, optional first arg is POJO of imports
* `camelCase` for helpers, modifiers, "control-flow" components is the new normal
* `link-to` positional args may have to be deprecated and/or made into a keyword?
* `has-block` vs `hasBlock`?

## TODO

* import * from https://github.com/ef4/rfcs/blob/js-compatible-resolving/text/0000-js-compatible-resolution.md;

* import * from https://github.com/emberjs/rfcs/blob/template-import/text/0000-template-imports.md;

## Summary

> One paragraph explanation of the feature.

## Motivation

> Why are we doing this? What use cases does it support? What is the expected
outcome?

## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## How we teach this

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
