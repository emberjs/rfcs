---
Start Date: 2014-09-30
RFC PR: https://github.com/emberjs/rfcs/pull/11
Ember Issue: https://github.com/emberjs/ember.js/pull/9527

---

# Summary

Improve computed property syntax

# Motivation

Today, the setter variant of CP's is both confusing, and looks scary as sin.
(Too many concepts must be taught and it is too easy to screw it up.)

# Detailed design

today:
------

```js
fullName: Ember.computed('firstName', 'lastName', function(key, value) {
  if (arguments.length > 1) {
    var names = value.split(' ');
    this.setProperties({
      firstName: names[0],
      lastName: names[1]
    });
    return value;
  }

  return this.get('firstName') + ' ' + this.get('lastName');
});
```

Tomorrow:
---------

```js
fullName: Ember.computed('firstName', 'lastName', {
  get: function(keyName) {
    return this.get('firstName') + ' ' + this.get('lastName');
  },

  set: function(keyName, fullName, oldValue) {
   var names = fullName.split(' ');

   this.setProperties({
     firstName: names[0],
     lastName: names[1]
   });

   return fullName;
  }
});
```


Notes:
------

* we should keep `Ember.computed(fn);`  as shorthand for getter only
* `get` xor `set` variants would also be possible.
* `{ get() { } }` is es6 syntax for `{ get: function() { } )`

Migration
---------

* 1.x support both, detect new behaviour by testing if the last arg is not null and typeof object
* 1.x+1 deprecate if last arg is a function and its arity is greater than 1


# Drawbacks

N/A

# Alternatives

N/A

# Unresolved questions

None
