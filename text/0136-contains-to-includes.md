---
Start Date: 2016-04-16
RFC PR: https://github.com/emberjs/rfcs/pull/136
Ember Issue: https://github.com/emberjs/ember.js/pull/13553

---

# Summary

[`contains`](http://emberjs.com/api/classes/Ember.Array.html#method_contains) is
implemented on `Ember.Array`, but [contains was renamed to includes in 2014]
(https://github.com/tc39/Array.prototype.includes/commit/4b6b9534582cb7991daea3980c26a34af0e76c6c)
- this proposal is for `contains` to be deprecated in favour of an `includes`
method on `Ember.Array`

# Motivation

Motivation is to stay in line with web standards

# Detailed design

First, implement `includes` polyfill in compliance with `includes` spec. Polyfill
sample from MDN is:

```js
if (!Array.prototype.includes) {
  Array.prototype.includes = function(searchElement /*, fromIndex*/ ) {
    'use strict';
    var O = Object(this);
    var len = parseInt(O.length) || 0;
    if (len === 0) {
      return false;
    }
    var n = parseInt(arguments[1]) || 0;
    var k;
    if (n >= 0) {
      k = n;
    } else {
      k = len + n;
      if (k < 0) {k = 0;}
    }
    var currentElement;
    while (k < len) {
      currentElement = O[k];
      if (searchElement === currentElement ||
         (searchElement !== searchElement && currentElement !== currentElement)) { // NaN !== NaN
        return true;
      }
      k++;
    }
    return false;
  };
}
```

Then, alias `contains` to `includes` with deprecation warning, deprecate in line with standard
deprecation process. I don't believe that adding the additional parameter will
have any affect on existing usage of `contains`.

# How We Teach This

* Update any references in docs and guides to `includes`
* Write a deprecation guide, mentioning any edge cases where the new `includes` behaves differently to `contains`, and giving migration examples
* Indicate in api docs that this is a polyfill

# Drawbacks

* May break existing apps
* [Was considered before but was too early](https://github.com/emberjs/ember.js/issues/5670#issuecomment-64084814)

# Alternatives

Keep current methods

# Unresolved questions

None
