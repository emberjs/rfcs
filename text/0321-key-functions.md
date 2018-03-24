- Start Date: 2018-03-24
- RFC PR: https://github.com/emberjs/rfcs/pull/321
- Ember Issue: (leave this empty)

# Key Functions for `each`/`each-in`

## Summary

This RFC proposes that we support passing functions to the `key` argument of the `each` and `each-in` helpers. These functions would be called for each item and would be expected to return they key value used in diffing.

For example,

```js
import Component from '@ember/component';

export default Component.extend({
  itemKey(item) {
    return `${item.factoryNumber}-${item.partNumber}`;
  }
});
```

```hbs
{{#each @items key=itemKey as |item|}}
  {{item-details item=item}}
{{/each}}
```

## Motivation

The `key` argument to `each` and `each-in` determines the key used in the Glimmer diffing algorithm. Here is a description of the currently accepted values?

`key` argument          | Computed key
------------------------|---------------------------------------------
`@identity`             | `guidFor(item)`
`@index`                | index of the item in the array / object keys
`@key` (`each-in` only) | the key of the key-value pair
other strings           | `get(item, path)`
otherwise               | error

If a key is not specified then `@identity` strategy is used.

Unfortunately, unless one of the default strategies works for your use case, you're stucking using a path on your object as the key. This can be problematic when you need to assemble a composite key from multiple values as in the example above. It is especially problematic when dealing with objects whose fields are out of your control (e.g. a moment date, an immutable.js record, a frozen object, etc.). Another less common reason you might want to use a key function is if your keys are expensive to compute and you want to cache them in a `WeakMap`.

## Detailed design

If `key` is a function, then Glimmer's diffing algorithm will call this function to determine the item's key.

For `each`, the function would be called with arguments `keyFn(item, index)`.
For `each-in`, the function would be called with arguments `keyFn(key, value)`.

The key function should not be invoked with a `this`.

One additional benefit of key functions is that they can be used to model the existing strategies in a uniform way:

`key` argument | `each` key function                | `each-in` key function
---------------|------------------------------------|-----------------------------------
`@identity`    | `(item, index) => guidFor(item)`   | `(key, value) => guidFor(value)`
`@index`       | `(item, index) => index`           | N/A
`@key`         | N/A                                | `(key, value) => key`
other strings  | `(item, index) => get(item, path)` | `(key, value) => get(value, path)`

## How we teach this

This feature just needs to be documented in the API docs and possibly the guides. We may want to reframe the existing (string-based) key documentation to be explained as shorthands for key functions.

## Drawbacks

There aren't any drawbacks I can see besides the usual concerns of increased file size and API surface. I believe that the increased flexibility of keys and making the built-in key strategies possible to be written in user space sufficiently offset these concerns.

## Alternatives

- Leave things as they are. I don't think this is a good idea.
- Think of an alternative way to solve this problem. I can't think of an option that is better than this.
