- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

To make static resolution of a dependency graph of components and helpers in
Handlebars templates possible at build time.

# Motivation

RFC #176 proposes a version of Ember with a "first-class system for importing
just the parts of the framework you need". Landing that RFC would make almost
all modularized code in the Ember ecosystem statically resolvable. The exception
being Handlebars templates where the `{{component}}` helper has been used.

This RFC is seeking to provide a system that can resolve usage of any component
and or helper for any given Ember application, by the means of the app author
declaring each component used in a template up front.

When implemented this RFC should unlock being able to tree-shake a full Ember
application.

As a secondary motivation, it should make it easier for an app author to trace
where a component and/or helper originates from.

# Detailed design

The implementation will rely on a syntax analogue to the ECMAScript 2015 syntax
for importing values from modules. Were in a JavaScript module importing another
module looks like this:

```js
import Ember from "ember";
import Analytics from "../mixins/analytics";

export default Ember.Component.extend(Analytics, {
  // ...
});
```

The proposed Handlebars analogue to this is:

```hbs
{{!import ./post-editor}}

<div>
  <h1>Editing: {{post.title}}</h1>
  {{post-editor post=post}}
</div>
```

The important bit is `{{!import ./post-editor}}`. This is seen by the current
Handlebars compiler as a comment and thus is ignored. The proposed
implementation would add meaning to this comment as being a import directive.
The choice for the import directives to be comments is a deliberate choice to
make the implementation backwards compatible with older implementations of the
Handlebars compiler.

This new directive does not have any output in the compiled result of the
template. It is only meant to be read by static-analysis tools.

Breaking down the import directive, it should start with the word `import`
followed by a path. This path can either be relative to the current
component/helper or absolute.

# How We Teach This

This system should be an opt-in feature to the Ember ecosystem. Proper guides in
the `README.md` of the tools coming forth from this RFC should be sufficient.

# Drawbacks

When opting in to this system, you will be adding a lot of directive comments to
templates.

# Alternatives

An alternative would be to  make the directives also have meaning at run-time.
Meaning that before a component is resolved by DI, the run-time would check a
map of imports to resolve the component/helper.

# Unresolved questions

- Should relative imports be relative to the component or to the template?
