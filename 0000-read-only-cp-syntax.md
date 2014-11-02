- Start Date: 2014-19-30
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Getter only computed properties should be readOnly by default

# Motivation

Overridable CP's are unexpected, hard to learn, and wanted in the minority of
cases. When it happens unexpeditly it is nearly always unwanted and the resulting
choas is hard to debug.

# Detailed design

```js
firstName: Ember.computed(function() {

}); // readOnly
```

Is equivalent to:

```js
firstName: Ember.computed({
  get: function() {
}); // readOnly (same as above)
```

```js
firstName: Ember.computed(function() {
  
}).overridable(); // writable
```
Is equivalent to:

```js
firstName: Ember.computed({
  get: function() {
}).overridable(); // writable
```

```js
firstName: Ember.computed({
  get: function() { },
  set: function() { }
}).overridable(); // throw new Error("Overridable cannot be used if a setter already exists....");
```

```js
firstName: Ember.computed({
  get: function() { },
  set: function() { }
}).readOnly();    // throw new Error("Cannot mark a CP as readOnly if it has an explicit setter");
```

```js

firstName: Ember.computed({
  get: function() { }
}); // on super class

firstName: Ember.computed({
  get: function() { },
  set: function(key, value) { this._super(key, value); }
}); // throw new Error("ReadOnly Error blah blah blah");
```

# migration:

* 1.x update internals to correctly use `writable` if needed (and runs tests against this)
* 1.x deprecate writability unless `writable` was explicitly called, or new `set` syntax  was used
* 2.0 flip the switch to `readOnly` by default and never look back.

# Drawbacks

N/A

# Alternatives

N/A

# Unresolved questions

N/A
