- Start Date: 2018-02-20
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# HTML Attribute and Property Rationalization

## Summary

This RFC's goal is to rationalize how to set attributes and properties of an `HTMLElement` as defined by a template. Today, we have a set of implicit rules of when to set an attribute using `setAttribute` and when we simply set a property on the element. For example:

```hbs
<div class="{{color}}">
  <p class="preview">{{firstName}}</p>
  <input value={{firstName}} onkeydown={{action "changedName"}} />
  <input value="Submit" type="Button" onclick={{action "submit"}} />
</div>
```

Will result in the following DOM operations if the context is `{ color: "red", firstName: "Chad" }`:

```js
let div = document.createElement('div');
div.setAttribute('class', 'red');
                                                 // <div class="red"></div>
let p = document.createElement('p');
p.setAttribute('class', 'preview');
let name = document.createTextNode('Chad');
p.appendChild(name);
div.appendChild(p);
                                                 // <div class="red"><p class="preview">Chad</p></div>
let input1 = document.createElement('input');
input1.value = 'Chad';
input1.onkeydown = () => {...};
div.appendChild(input1);
                                                 // <div class="red">
                                                 //   <p class="preview">Chad</p>
                                                 //   <input />
                                                 // <div>
let input2 = document.createElement('input');
input2.setAttribute('value', 'Submit');
input2.setAttribute('type', 'button');
input2.onclick = () => {...};
div.appendChild(input2);
                                                 // <div class="red">
                                                 //   <p class="preview">Chad</p>
                                                 //   <input />
                                                 //   <input value="Submit" type="button" />
                                                 // <div>
```

This illustrates the setting inconsistencies based on the attribute name and if the value is dynamic or not. To remove these inconsistencies we would like to move to the [HTML semantics](https://html.spec.whatwg.org/multipage/dom.html#attributes) by always setting attributes. At the same time we will introduce a `{{prop}}` and `{{on}}` modifier for explicitly setting properties and events on `HTMLElement`s. The above template would be transformed to look like the following to retain the semantics of today:

```hbs
<div class="{{color}}">
  <p class="preview">{{firstName}}</p>
  <input {{prop value=firstName}} {{on keydown=(action "changedName")}} />
  <input value="Submit" type="Button" {{on click=(action "submit")}} />
</div>
```

By moving to these new semantics we will shore up long standing confusion in this area, unify HTML serialization and DOM construction, and open the door for first class support for custom elements.

## Motivation
This proposal:

- brings clarity and [addresses confusion](#appendix-a) as how to set a property on an element
- enables the usage of [custom elements](https://custom-elements-everywhere.com/)
- fixes HTML serialization issues for server-side rendering
- removes the need for `TextSupport` mixin for supporting input events
- removes the need for `{{textarea}}` and `{{input}}`

## Detailed design

To move to this more explicit syntax in a backwards compatable we will convert well-known properties to use the `{{prop}}` and `{{on}}` modifiers via a codemod and retain the runtime setting rules for `attributeBindings`.

### Transforming Through Codemod
Due to the declarative nature of HTML we are able to see exactly which attributes are being set on which elements. This allows us to understand where we need to set properties and where we can safely rely on setting attributes. Using the list of [HTML attributes](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes) and [WAI-ARIA attributes](https://www.w3.org/TR/wai-aria-1.1/) we can then determine which attributes in templates need to be converted to use either `{{prop}}` or `{{on}}`.

### Runtime Rules
With Ember Components we have the notion of `attributeBindings` to set attributes on the wrapping element as defined by the `tagName` on the component class. For this case we will continue to use the setting rules we have today. Notably, [Glimmer Components](https://glimmerjs.com/guides/templates-and-helpers) have adopted the "Outer HTML" semantics long ago and the experience has been very positive. This means that when Glimmer Components become available in Ember they would be covered by the codemod as described above.

Furthermore, we would deprecate the usage of `{{textarea}}`, `{{input}}`, and the `TextSupportMixin`. This will require a deprecation guide for showing developers how to convert to use these new primitives.

## How we teach this
The vast majority of the time you want to be setting attributes on elements. The cases in which you need to set properties are when you are setting a dynamic value to an element that takes user input e.g. `<input />`, `<option />`, and `<textarea />`. For example we automatically would set the `value` prop on `<input />` instead of setting the attribute:

```hbs
<input value={{myValue}} />
```

While the codemod will re-write existing instances of this, we can warn developers in development builds that they should set the value using the `{{prop}}` modifier. We can also expand the rules of [Ember Template Lint](https://github.com/rwjblue/ember-template-lint) to be a linting error for these limited and well known cases. The guides should be updated calling out these examples and should be referenced in any warning message that is raised.

We should also add a section to the docs about how to interop with native web components as this unlocks the primitives to properly support them.

## Drawbacks
The drawback of this is that we are making developers know about some nuanced details of the updating semantics of HTMLElements. The tradeoff is that serve-side rendering works as expected and developers have a clear path for adopting natvie web components.

## Alternatives
There are a couple different alternatives. We could keep `{{textarea}}` and `{{input}}` around and somehow special case them as special components that always set properties. However, this means we would have to expand helper support to things like `<option />`. It's also odd to be creating primitives just for the sake of updating semantics.

The other option is to take Angular's approach and always set properties. However, this creates the inverse problem from serve-side rendering. Since you can slot an Element instance with any field it is unclear how we serialize any attribute. This means we would have to do all attribute setting during server-side rendering and then set properties on the client, but this may surface even more issues.

## Unresolved questions

TBD?

## Appendix A

Short list of issues related to property vs. attribute setting:

- https://github.com/emberjs/ember.js/issues/16171
- https://github.com/emberjs/ember.js/issues/11678
- https://github.com/emberjs/ember.js/issues/15709
- https://github.com/emberjs/ember.js/issues/13628
- https://github.com/emberjs/ember.js/issues/13484