- Start Date: 2016-04-03
- RFC PR:
- Ember Issue:

# Summary

Introduce a `uniqBy` function in the Enumerable mixin.

` collection.uniqBy('name')`

# Motivation

The Enumerable mixin already provides `filterBy`, `sortBy`, `mapBy`, `rejectBy`, and `findBy`. It seems counterintuitive that the user has to write out the logic that would implement a `uniqBy` when the existing api conventions imply that it at the very least *could* exist.

Also, the `uniq` function that does exist is limited to only working with primitive data types. A `uniqBy` would extend application of the method to cover objects as well.

# Detailed design

```js
*/
@method uniqBy
@param {String} property name(s) to uniqify on
@return {Ember.Enumerable}
@public
*/
uniqBy() {
  var uniqKeys = arguments;
  var ret = emberA([]);

  this.forEach((item) => {
    var options = ret;

    for (var i = 0; i < uniqKeys.length; i++) {
      var key = uniqKeys[i];

      options = options.filter(iter.apply(options, [key, item[key]]));
    }

    if (options.length <= 0) {
      ret.push(item);
    }
  });

  return ret;
},
```

Or something similar.
# How We Teach This

We would need additional documentation on the Ember.Enumerable class documentation, but otherwise it should not require any additional efforts to teach.

# Drawbacks

None that I'm aware of - perhaps poor performance implications when called on enumerables of great length? But this would be a drawback of the implementation, and not so much a commentary on the idea itself.

# Alternatives

The alternative is to continue having the users of Ember write their own functions to return unique values. Which, while not terrible, would be less convenient than if Ember already provided it.

# Unresolved questions

None that I know of.
