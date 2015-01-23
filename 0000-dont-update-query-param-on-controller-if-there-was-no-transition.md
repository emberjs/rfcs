- Start Date: 2015-01-03
- RFC PR:
- Ember Issue:

# Summary

When a property used in a query param is changed and there is no transition to
the other route, don't update the query param with a value from the URL.

# Motivation

When you use query params for live filtering, it may be necessary to throttle a
filtering function to avoid running costly computation too often. For example
you may want to use an input field to filter by a string and send a change event
every 200ms at most.

An example code could look like that:

```javascript
Ember.Controller.extend({
  queryParams: ['filter'],

  filteredItems: function() {
    // costly computation
   return items;
  }.property('filter'),

  updateFilter: function() {
    var value = this.get('lastFilterValue');
    this.set('filter', value);
  },

  actions: {
    updateFilter: function(value) ->
      this.set('lastFilterValue', value);
      Ember.run.throttle(this, 'updateFilter', [], 200, false);
    }
  }
});
```

As you can see, the `updateFilter` method is throttled. This doesn't work really
well with query params, because changing the filter's value triggers a
transition, which in effect will update the filter's value again.

Imagine that a user writes 2 characters "fo" during 200ms and then another
character "o" during the next 200ms. The `updateFilter` method should be called
twice: once with "fo" and second time with "foo". As a result URL should contain
`?filter=foo` and the `filter` property should be set to `foo`. It doesn't
work like that, however. After the `updateFilter` method is called for the first
time the router starts a transition. In the meantime the user enters the third
character "o". If the transition finishes after the character was entered, but
before another 200ms passed, `filter`'s value will be set to whatever was passed
to a transition, "fo" in this case. The user will end up without one of the
entered letters.

# Detailed design

There are two possible solutions to this problem:

1. If the transition is triggered by a query param change, ie. it stays in the
   same route, don't actually transition, just change the URL.
2. If the transition is triggered by a query param change, don't update the
   query param after the transition is made, but rather use the value that's
   already in the controller.

# Drawbacks

I'm not sure at this point, I'll appreciate any feedback.

# Alternatives

I can't think of any alternatives other than implementing one of the proposed
solutions in the application.

# Unresolved questions

* I'm not sure if avoiding transition can lead to substantial differences. It
  would be greate to have a list of hooks that are called currently
