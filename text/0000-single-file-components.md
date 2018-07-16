- Start Date: 2017-05-22
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

It is currently possible to create single file Ember components with component
templates declared inline using a tagged template string assigned to the `layout`
property. However, this pattern is not documented and support is unknown. This RFC
proposes documenting support for this pattern.

# Motivation

The benefits of same file template declarations are mostly development conveniences
focused around:
- All component concerns available in a single file, making tracking property or
  service usage throughout a component easier.
- Easier component file portability when restructuring components/creating namespaces
  for components.

Anecdotally our team has also found that single file components support focused
modular design, as it's easy to 'feel' when your component is trying to address too
many concerns as the file length grows.

Primarily though, single file components are a convenience that makes working with
numerous components at once easier to manage. I think this is a benefit of Vue and
React that is already available in Ember as well.

# Detailed design

This pattern is currently possible by assigning a tagged template string to the
component layout property:

```javascript
import Component from 'ember-component';
import inject from 'ember-service/inject';
import hbs from 'htmlbars-inline-precompile';

export default Component.extend({
  cart: inject()

  passedItemPrice: 0,

  layout: hbs`
    <div>
      <span>Item cost is: {{total}}</span>
      <button {{action 'removeFromCart' target=cart}}>Remove</button>
    </div>
  `
})
```

For proper syntax highlighting of handlebars inside the tagged template literal our
team has created highlighters for
[Sublime](https://github.com/healthsparq/sublime-ember-syntax),
[Atom](https://atom.io/packages/language-ember), and
[VSCode](https://marketplace.visualstudio.com/items?itemName=dhedgecock.ember-syntax).

A proposed change to this pattern is allowing the component template to be declared
using property `template` instead of `layout`, as it would likely be easier for newer
users to remember.

According to Robert Jackson, the existing pattern should be ok in production, and
hopefully still is after the Glimmer 2 rewrites (we are using it in production at
HealthSparq with no issues currently).
Quoted: [Is it okay to use for components in production?](https://github.com/ember-cli/ember-cli-htmlbars-inline-precompile/issues/26)

Work needed would be:
- Updating Ember guides documentation for declaring component templates.
- Creating a new component blueprint for inline template components.

# How We Teach This

Our team has been referring to this pattern as 'inline template declaration'. The
syntax of declaring a template using a tagged template string can be documented in
the Templates section of the Ember guides, perhaps in a new section called 'Declaring
Templates Inline'. It could be presented as an alternate to defining a component's
template using a separate .hbs file.

Although this would change learning Ember as a new user, I personally think that it
would make learning Ember easier if it is used as the primary pattern for creating
components. Although the component generator creates the component and template files
in the correct file locations, it may not be immediately clear to a new user that
the file structure is required, and why the structure is required. Assigning a
template to the `layout` (or `template`) property doesn't require understanding file
lookups based on directory location.

# Drawbacks

1. In terms of teaching Ember, some might think that this complicates learning Ember
  by giving new users two choices for how to declare a component's template.
2. Although syntax highlighting for Sublime/Atom/VSCode exists, users must install
  a package/extension to get highlighting.

# Alternatives

1. Officially not support single file component/template definitions.
2. Support single file component/template definitions with a pattern different from
  the tagged template string that works today.

#### Other framework solutions:
- Templates declared using property `template` and strings (Vue).
- Templates declared with JSX (React).

#### Impact of not doing:
Missed developer convenience where other frameworks offer single file
component+template patterns and Ember does not.

# Unresolved questions

1. Can this pattern be supported when new Glimmer components land?
