- Start Date: 2015-02-21
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Aims to provide a centralised property on components for exposure and definition of attributes.

# Motivation

Components currently have the issue of their template usage defining the meaning of the components attributes. This RFC aims to set out a method of specifying the clear interface of a component.

- Accidental use of an incorrect attribute would be caught whilst writing the template
- Knowing the complete set of attributes becomes problematic:
  - Components that are extended may add new attributes
  - Components defined in an Arden may receive more attributes in the extended in app/template/components/componentname
- Validating a components usage is tricky as there is no list of valid attributes
- Components become more self documenting by having a simple property to look for which defines the permitted attributes
- Adding typing to a components attributes would provide meaning to each attribute
  - Reduces the need for computed properties as attributes could be transformed into typed property data

# Detailed design

## Component data types
Exposed via `Ember.Component.attr` is a function to define the types of the component. By specifying an attribute to be of a certain type Ember will be able to transform the attribute into more useful data.

The API should match that of [Ember Data attr](http://emberjs.com/api/data/#method_attr). When data comes into the component via the attribute the data will be transformed by the same transformations used by Ember data.

## AttrTypes

By adding `attrTypes` key to a component:

- Adding a attribute in the template that is not in the list will trigger an exception in development mode.
- Component template is only passed properties that exist in the attrTypes object.

By omitting `attrTypes` key to a component:

- Adding any attribute to the component doesn't cause an exception.
- Component template is exposed all properties passed and computed properties.

## Examples

### Valid use

my-component.js
```
import Ember from 'ember';
var attr = Ember.Component.attr;
 
export default Ember.Component.extend({
  attrTypes: {
    'other-name': attr('string'),
    exciting: attr('boolean'),
  }
});
```

my-template.hbs
```
  {{#my-component other-name="thing" exciting="1" }}
  {{/my-component}}
```

my-component.hbs
```
  name: {{other-name}}
  {{#if exciting}} {{!-- exciting === true --}}
      Wow exciting!
  {{/if}}
```
### Invalid use

my-component.js
```
App.MyComponent = Ember.Component.extend({
  attrTypes: {
    'other-name': attr('string')
  }
});
```

my-template.hbs
```
  {{#my-component the-name="thing"}}
  {{/my-component}}
```
Raises error: 'the-name' is not in the attrTypes for the component

# Drawbacks

Beginners could be confused why sometimes template usage doesn't work as expected for an addon.

Mitigate the issue by triggering an error when the incorrect usage happens, template authors should notice the issue and resolve.

# Alternatives

- An implementation similar to or the same as: [definedProperties](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperties)


# Unresolved questions

- If a separate property `exposedProperties` makes sense to separate out the permitted and template exposure.
