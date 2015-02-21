- Start Date: 2015-02-21
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Providing a centralised property on components for exposure and definition of properties.

# Motivation

Components currently have the issue of their template usage defining their meaning.

- Accidental use of an incorrect parameter would be caught as writing the template
- Knowing the complete set of parameters becomes problematic:
  - Components that are extended may add new attributes
  - Components defined in an addon may receive more attributes in the extended in app/template/components/componentname
- Validating a components usage is tricky as there is no list of valid attributes
- Components become more self documenting by having a simple property to look for which defines the permitted properties

# Detailed design

By adding `allowedProperies` key to a component:

- Adding a property in the template that is not in the list will trigger an exception.
- Component template only is passed properties in the array of strings.

By omitting `allowedProperties` key to a component:

- Adding any property doesn't cause an exception.
- Component template is exposed all properties passed in and computed properties.

## Examples

### Valid use

my-component.js
```
App.MyComponent = Ember.Component.extend({
  allowedProperties: [
    'other-name'
  ]
});
```

my-template.hbs
```
  {{#my-component other-name="thing"}}
  {{/my-component}}
```

my-component.hbs
```
  name: {{other-name}}
```
### Invalid use

my-component.js
```
App.MyComponent = Ember.Component.extend({
  allowedProperties: [
    'other-name'
  ]
});
```

my-template.hbs
```
  {{#my-component the-name="thing"}}
  {{/my-component}}
```
Raises error: 'the-name' is not in the allowedProperty list for the component

# Drawbacks

Beginners could be confused why sometimes template usage doesn't work as expected for an addon.

Mitigate the issue by triggering an error when the incorrect usage happens, template authors should notice the issue and resolve.

# Alternatives

- An implementation similar to or the same as: [definedProperties](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperties)


# Unresolved questions

- If a separate property `exposedProperties` makes sense to separate out the permitted and template exposure.
- Typing of properties (potentially related to: https://github.com/emberjs/rfcs/pull/29)
