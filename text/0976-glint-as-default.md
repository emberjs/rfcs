---
stage: accepted
start-date: 2023-09-29T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - framework
  - learning
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/976
project-link:
suite:
---

# Enable Glint by Default

## Summary

[Glint](https://typed-ember.gitbook.io/glint) should be enabled as a default for Ember projects.

This RFC proposes adopting Glint as a default/recommended tool in the Ember.js framework. Glint is a static TypeScript-based template type checker that aims to improve developer experience and catch template-related errors at build time. By incorporating Glint into Ember.js, we can enhance type safety, provide better tooling, and encourage best practices when working with templates.

Adopting Glint as a default/recommended tool in Ember.js is a step towards improving developer experience, reducing errors, and promoting best practices in Ember.js template development. By integrating Glint and providing guidance, Ember.js can stay at the forefront of modern web development practices.

## Motivation

> Why are we doing this? What use cases does it support? What is the expected
outcome?
>

Ember.js has a strong focus on developer productivity and maintainability. Templates are a critical part of Ember applications, and ensuring their correctness and type safety is essential. Currently, the Ember template system relies on runtime checking for template-related issues, which can lead to runtime errors and decreased developer productivity. By adopting Glint, we can address these issues and provide several benefits:

**Static Typing:** Glint uses TypeScript to perform static analysis on templates, enabling early detection of type-related errors in templates, such as incorrect variable usage and undefined variables.

**Improved Tooling:** Glint offers superior tooling support for templates, including code editors' autocomplete and type checking, which can enhance the development experience for Ember developers. We can leverage this tooling even in JavaScript-only projects.

**Compile-time Checks:** With Glint, template-related errors can be caught at build time, significantly reducing the likelihood of runtime issues and improving application reliability.

**Best Practices:** Encouraging the use of Glint can promote best practices in Ember.js development, leading to more maintainable and robust codebases.

## Detailed design

> This is the bulk of the RFC.
>

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.
>

The adoption of Glint as a default/recommended tool in Ember.js involves the following steps:

**Integration:** Add `glint` as a dependency by default, allowing Glint to analyze templates during the build process for TypeScript projects and enabling Glint tooling in both TypeScript and JavaScript projects. Configure `glint` by default in blueprint output. Currently, `glint` is *mostly* configured when running the blueprint with the `--typescript` flag.

**Community Adoption:** Encourage the Ember.js community to adopt Glint by showcasing its benefits and providing resources for migration.

**Plugin Ecosystem:** Encourage the development of plugins and extensions that enhance Glint's functionality for Ember.js projects. ([VSCode extension](https://marketplace.visualstudio.com/items?itemName=typed-ember.glint-vscode), other IDE extensions).

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?
>

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?
>

> How should this feature be introduced and taught to existing Ember
users?
>

### Terminology

********************registry:******************** In traditional templates with .hbs files (i.e. not template tag), all lookups are done via strings. Since we can’t rely on import statements to discover types, we instead use a type-registry. The type-registry serves to map strings to their corresponding type definitions.

```tsx
declare module '@glint/environment-ember-loose/registry' {
	export default interface Registry {
		'MyComponent': typeof MyComponent;
	}
}
```

### Documentation

**Initial Announcement:** Make an official announcement on the Ember.js blog and mailing list to inform the community about the plan to adopt Glint as a default/recommended tool.

**Documentation Updates:** Update the Ember.js Gudies to include information about Glint, its benefits, and how to use it effectively.

Provide examples and best practices for leveraging Glint effectively. The current [TypeScript Guides](https://github.com/ember-learn/guides-source/pull/1960) have many places where we can reference Glint (e.g. in areas where we currently say that templates are *not* type-checked). We can also reference the [existing Glint documentation](https://typed-ember.gitbook.io/glint/) and merge its relevant content with the TypeScript guides. Additionally we should document the existence of any plugins/extensions as mentioned above in “Plugin Ecosystem.”

**Integration Work:** Collaborate with the Glint project (which is maintained by Ember’s TypeScript and Tooling Core Teams) to ensure smooth integration into Ember.js, addressing any compatibility issues that may arise.

**Community Outreach:** Organize blog posts and conference talks to educate the Ember.js community about Glint and its advantages.

## Drawbacks

> Why should we not do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.
>

> There are tradeoffs to choosing any path, please attempt to identify them here.
>

There are some potential drawbacks to adopting Glint:

**Learning Curve:** Developers who are not familiar with TypeScript may face a learning curve when integrating Glint into their Ember projects.

**Migration Effort:** Existing projects may require effort to migrate from not having a template checking system to Glint.

**********************************************Registries Boilerplate:**********************************************

Because Ember's template resolution occurs dynamically at runtime when using `.hbs` files, Glint needs a way of mapping the names used in your templates to the actual backing value they'll be resolved to. This takes the form of a "type registry" as defined above.

Note that this is *not* an issue for users using `.gts` templates.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?
>

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.
>

The primary alternative to adopting Glint is to continue using Ember without a template type-checking system. However, this would mean missing out on the benefits of static type checking and improved tooling provided by Glint.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
>

### Is there redundancy between Ember Language Server and Glint Language Server?

We should verify that there are no redundancies between the Glint Language Server and the Ember Language Server used in IDEs. If there is, we should answer the following questions:

Should Glint Language Server features be merged into the Ember Language Server? Alternatively, should we remove the redundancy between them but keep them separate?

### Does the Glint Language Server need full feature parity with the TypeScript Language Server before this is merged?

Currently, we recommend Glint users enable the Glint Language Server in their IDE and disable the TypeScript Language Server to avoid duplicate errors. The Glint Language Server does not, however, currently have full feature parity with the TypeScript Language Server, so in doing so, users are losing some language server features. There is [work in progress](https://github.com/typed-ember/glint/issues/626) to rectify this issue.

### Should we require that GTS is recommended before we recommend Glint?

As noted above, using Glint with `.hbs` files, there is boilerplate necessary for Glint to recognize component invocations in the templates. With `.gts` files, this is not an issue.

In the opinion of the authors of this RFC, the benefit of Glint outweighs the downsides of requiring the boilerplate.

Additionally, `.gts` is not supported for route templates by default, so that issue would need to be resolved, delaying the Glint recommendation further.

### Should we include JSDoc types in the JS blueprints?

With JSDoc component signatures, JS users can also get many of Glint’s benefits.
