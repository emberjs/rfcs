- Start Date: 2018-03-25
- RFC PR: https://github.com/emberjs/rfcs/pull/323
- Ember Issue: (leave this empty)

# Array functions

## Summary

This RFC proposes a way to work with arrays in ember without requiring that we extend the global `Array` prototype or manually wrap arrays with `Ember.A(array)`.

Specifically, for each method on the `MutableArray` mixin we will provide a similarly named function that takes an array as its first argument followed by the arguments to the method.

For example, instead of calling `array.pushObject(item)` you can call `pushObject(array, item)`. For a full list see the expandible list in [Detailed design](#detailed-design).

## Motivation

Currently it is not a great experience to write an app with `Array` prototype extensions disabled. You end up needing to wrap arrays everywhere with `Ember.A(array)` which loops through each method on the `NativeArray` mixin and assigns it as an enumerable own property on the array.

The are several problems here:

- It's annoying to defensively write `Ember.A()` around every array
- It's wasteful to looping through all the array methods and assign them all. Especially if you're only going to use one of them
- Iterating over an array with `for..in` or `Object.keys` will not behave as generally expected because the methods are assigned enumerably (for performance reasons)

All of these problems go away when using array functions instead of array methods.

It's also worth noting that when you using fastboot you are required to have `Array` prototype extensions disabled currently and thus forced to deal with these issues.

## Detailed design

For each method in the `MutableArray` mixin we will export a similarly named function from `@ember/array`.

For example, the method `array.objectAt(index)` will map to the function `objectAt(array, index)` with implementation

```js
function objectAt(array, index) {
  if (Array.isArray(array)) {
    return array[index];
  } else {
    return array.objectAt(index);
  }
}
```

As you can see, the native array case is handled explicitly in order to avoid assuming that the global `Array` prototype has been extended, but otherwise we just delegate to the `objectAt` method.

For reference, below is an expandable list of all array functions we will need.

<details>
  <summary>List of array functions (49)</summary>
  <code>
    <ul>
      <li>addArrayObserver</li>
      <li>addObject</li>
      <li>addObjects</li>
      <li>any</li>
      <li>arrayContentDidChange</li>
      <li>arrayContentWillChange</li>
      <li>clear</li>
      <li>compact</li>
      <li>every</li>
      <li>filter</li>
      <li>filterBy</li>
      <li>find</li>
      <li>findBy</li>
      <li>forEach</li>
      <li>getEach</li>
      <li>includes</li>
      <li>indexOf</li>
      <li>insertAt</li>
      <li>invoke</li>
      <li>isAny</li>
      <li>isEvery</li>
      <li>lastIndexOf</li>
      <li>map</li>
      <li>mapBy</li>
      <li>objectAt</li>
      <li>objectsAt</li>
      <li>popObject</li>
      <li>pushObject</li>
      <li>pushObjects</li>
      <li>reduce</li>
      <li>reject</li>
      <li>rejectBy</li>
      <li>removeArrayObserver</li>
      <li>removeAt</li>
      <li>removeObject</li>
      <li>removeObjects</li>
      <li>replace</li>
      <li>reverseObjects</li>
      <li>setEach</li>
      <li>setObjects</li>
      <li>shiftObject</li>
      <li>slice</li>
      <li>sortBy</li>
      <li>toArray</li>
      <li>uniq</li>
      <li>uniqBy</li>
      <li>unshiftObject</li>
      <li>unshiftObjects</li>
      <li>without</li>
    </ul>
  </code>
</details>

## How we teach this

This functions must obviously be documented under the `@ember/array` package in the API docs.

We should encourage people to use the phrase "array functions" to describe these functions and "array methods" to describe the method versions.

We should encourage their usage in addons and fastboot apps instead of using `Ember.A`.

In the future, we may want to encourage their usage in all apps so that we can deprecate extending the global `Array` prototype, but that is out of scope for this RFC.

## Drawbacks

This RFC proposes a considerable increase in API surface area however it's mostly duplicating an existing, well known API so it does not increase the mental burden very much.

I also expect the total number of bytes added to the framework be fairly small since it is mostly just shuffling around existing code.

## Alternatives

- We could not do this and continue using `Ember.A` in addons / fastboot apps

I don't think this is a good path for the reasons listed in [Motivation](#motivation),

- Improve fastboot so that it allows prototype extensions

It's true that this would writing fastboot apps more ergnomic but it doesn't solve the problem for adddon authors and it doesn't solve the greater problem of getting us closer to Ember not rely on global prototype extensions for a good user experience.
