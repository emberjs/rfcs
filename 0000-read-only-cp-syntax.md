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


```js
firstName: Ember.computed(function() {
  
}).writable(); // writable
```

```js
firstName: Ember.computed(function(key, value) {

}); // writable
```

```js
firstName: Ember.computed(function(key, value) {

}).readOnly(); // readOnly but why?
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
