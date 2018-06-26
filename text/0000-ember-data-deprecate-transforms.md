- Start Date: 2018-06-24
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# (Ember Data Deprecate Transforms)

## Summary

> This RFC suggests deprecating Ember Data [transforms](https://www.emberjs.com/api/ember-data/release/classes/DS.Transform).

Example:

```javascript
// app/models/person.js
import DS from 'ember-data';

export default DS.Model.extend({
  name: DS.attr('string'), //<--
  age: DS.attr('number'), //<--
  admin: DS.attr('boolean'), //<--
  birthday: DS.attr('date') //<--
});
```

## Motivation

> Ember Data `transforms` have, for some time, been the cause of confusion for Ember Data users. Anything that can be done with `transforms` can also be done using Serializers.
>
> Part of the confusion was aided by the [no-empty-attrs](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-empty-attrs.md) ESLint Plugin Ember Rule. This rule was included as a recommended rule at one point in time in the plugin.
>
> `Transforms` are not *types*. They allow serializing and deserializing model `attrs`. However their use often causes people to assume that they are *types*.
>
> Another issue, `transforms` can have a bad effect on performance when they are used incorrectly, or at all.
>
> tl;dr `Transforms` are confusing and don't provide any added value that can't be achieved using serializers.

## Detailed design

> `Transforms` will be deprecated and any use of `transforms` will cause a deprecation warning, notifying the user that this method is deprecated and providing a link to an explanation of how to achieve the same outcome with serializers.
>
> An Ember Addon can be built to help any current users of `transforms` migrate to not using them over time.

## How we teach this

> We need to remove mentions of `transforms` in the Ember Guides and API:
> [Defining Models](https://guides.emberjs.com/release/models/defining-models/#toc_transforms),
> [Customizing Serializers](https://guides.emberjs.com/release/models/customizing-serializers/#toc_creating-custom-transformations),
> [API DS.Transform](https://www.emberjs.com/api/ember-data/release/classes/DS.Transform)
>
> Instead of these we will need to teach how to do the same things that were possible with `transforms` but with serializers.
>
> Once we remove the use of `transforms` from the guides and instead explain how to use serializers to achieve the same result, new users will learn to use serializers and not `transforms`.

## Drawbacks

> Currently `transforms` are being used and taught and deprecating them will remove a long used feature of Ember Data.
>
> We will need to write up more information on serializers to cover for the lack of `transforms`. However this isn't necessarily a bad thing.

## Alternatives

> We could not do this and just leave `transforms` as part of Ember Data. We could try to educate users about the actual uses of `transforms` and remove any notions of using them as `attr` *types*.

## Unresolved questions

> ?
