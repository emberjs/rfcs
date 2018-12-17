- Start Date: 2018-10-11
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

This RFC modifies the Module Unification RFC (#143) to use standard JavaScript imports in templates rather than Ember-specific two-level lookup semantics. It also scopes RFC #367 down to describing packages in dependency injection APIs, and supersedes the mechanism in that RFC for importing components into templates.

# Motivation



# Detailed design

## Phase 1: Components and Helpers

Templates can now have "front-matter", sandwiched between two `---` at the top of the template. The front-matter is required to start on the first line of the template, and the first line must literally be `---` followed by a newline.

```hbs
---
import { Header } from "./header";
import { Button } from "$components/button";
import PowerSelect from "ember-power-select";
import { TabBar } from "@gadgets/ui";
---

<TabBar @tabs={{this.tabList}}>

<Header>Hello visitors</Header>
<Button>I agree to the terms</Button>

<PowerSelect @options={{names}} @onchange={{action "foo"}} as |name|>
  {{name}}
</PowerSelect>
```

The syntax of the front-matter is a subset of ECMAScript 2018: a list of imports sandwiched between two `---`. Formally, the syntax is:

```
FrontMatter :
  "---" LineTerminator FrontMatterList LineTerminator "---"

FrontMatterList :
  ImportDeclaration
  FrontMatterList ImportDeclaration
```

### Component Implementation Strategy

In Phase 1, imports affect two invocation forms: `{{importedName}}` and `<ImportedName>`.

In the classic directory structure, when the compiler would consult the resolver to identify a helper or component, it will first check to see whether the name is the same as an import. If it is, the compiler will resolve the component or helper to the location specified by the import statement.

From an implementation perspective, when the Glimmer compiler sees something that looks like a component name, it eventually invokes `lookupComponentDefinition` on the runtime resolver provided by Ember. Today, that code path uses the Ember resolver to resolve the name into the component class and template.

In Phase 1, imports will be added to the template meta of a template, and the runtime resolver will get the component class and layout using the module system directly instead of working through the container system.

In order to maintain support for component injections, we will need a way to instantiate the component class through the container (as a `component`) even though we didn't use `lookup` or `lookupFactory`. It should be possible to create [`FactoryManager`][factory-manager] in Ember's container to handle this use-case.

[factory-manager]: https://github.com/emberjs/ember.js/blob/f73d8440d19cf86a10c61ddb89d45881acfcf974/packages/@ember/-internals/container/lib/container.ts

### Helper Implementation Strategy

It would also be possible to import helpers in the same way. The implementation strategy is similar to the strategy for components. The `lookupHelper` hook on the Ember runtime resolver (which also takes template meta), would look up helper names in the module system rather than through the resolver. Class-based helpers would be instantiated similarly to components.

### Helper or Component Ambiguity

Unlike module unification, this RFC does not require that helpers are named `helper` in order to be used. As a result, any name that is imported in a template might be a helper or it might be a component.

For the initial implementation, we will use the brand installed by the `helper` function in `@ember/component/helper` to disambiguate between helpers and components. This means that if `lookupHelper` is called by Glimmer, before returning the value found by the *Helper Implementation Strategy* above, `lookupHelper` will first do a brand check that verifies that the value it found is indeed a helper. If not, it will return `null`. Similarly, `lookupComponentDefinition` will return `null` if the value found is branded as a helper.

> Note: In the longer-term, we will want to use static information (such as decorators or other syntactic patterns) to disambiguate between helpers and components during precompilation. Since we are not currently in a position to use the Bundle Compiler in Ember, we can defer identifying the exact static form for now.

### Module Unification Interaction

### Component or Helper Values in JavaScript

### `$components`

## Phase 2: Arbitrary Values

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## Linting

# How We Teach This

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

How should this feature be introduced and taught to existing Ember
users?

# Drawbacks

Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

There are tradeoffs to choosing any path, please attempt to identify them here.

# Alternatives

What other designs have been considered? What is the impact of not doing this?

This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?
