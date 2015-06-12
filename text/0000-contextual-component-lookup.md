- Start Date: 2015-06-12
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Currently, you can only invoke components using a global name. The goal of this
RFC is to allow better composition of components by allowing components
to be referenced through property lookups on the current scope. For example,

```hbs
{{this.localInput value=post.title}}
```

where `localInput` is a property whose value is a component factory.

# Motivation

The primary motivation is to support robust communication channels between
components, typically between a parent component and its children. The current
best paractice is for a child component, upon creation, to walk up the view
tree to find it's parent using `nearestOfType`.

Sometimes `nearestOfType` is exactly what you want, but often times it is not.
Usuaully, you'd just prefer to have the component be instantiated with a
reference to its parent. Consider a case with nested components:

```hbs
{{#parent-component}}
  {{#parent-component}}
    {{child-component}}
  {{/parent-component}}
{{/parent-component}}
```

In this case, `nearestOfType` can't be used to hook the child component up the
outer parent component. This is a similar problem that existed with context-
changing `{{each}}`, which block params solved.

## `form-for` Example

This section describes an example of how you'd use this technique to
yield contextualized components in a form building component.

Let's say we want our users to write

```hbs
{{#form-for post as |f|}}
  {{f.input 'title'}}
  {{f.input 'body'}}
  {{f.submit}}
{{/form-for}}
```

then our component template and class could look like

```hbs
{{! templates/components/form-for.js }}

{{yield controls}}
```

and

```js
// components/form-for.js

import Ember from 'ember';
import { curry } from 'factory-utils';

export default Ember.Component.extend({
  controls: Ember.computed(function() {
    var inputFactory = this.container.lookupFactory('component:form-for-input');
    var submitFactory = this.container.lookupFactory('component:form-for-submit');

    return {
      input: curry(inputFactory, { form: this }),
      submit: curry(submitFactory, { form: this })
    };
  })
});
```

where `curry` is a simple helper which wraps the factory and assigns the extra
properties to the instance.

In the future we can use the injection API to avoid directly accessing the
container. 

# Detailed design

Any component invocation with a dot separator `.` in the name of the component
will perform a lookup of this property on the scope. The value must be a factory
and the factory must create component instances, otherwise an error is thrown.

The affected syntaxes are:

```
{{this.component}}
{{this.component name=name}}
<this.component />
<this.component name="{{name}}" />
```

# Drawbacks

In the HTML component case, it feels a little weird that the tag name does a
property lookup without being surrounded curlies. That said, I like the simplicity.

An alternative syntax (that I'm not really a fan of) is

```hbs
<{{this.component}} name="{{name}}" />
```


The requirement for the dot separator may also be seen as a drawback. The reason for
enforcing the dot separator is to avoid accidental fallback to global components.

This problem seems unavoidable as long as `{{foo}}` is ambiguous between a property
lookup and a helper call.

# Alternatives

I don't know of any alternative strategies that support contextualized components.

# Unresolved questions

- Should this also be available for helpers?
