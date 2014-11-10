- Start Date: 2014-11-09
- RFC PR: 
- Ember Issue: 

# Summary

Filter Object hold state used to filter records in the Ember Data
store.

# Motivation

The current Ember Data API for filtering the store encourages memory
leaks by creating many `FilteredRecordArray`s which are not reused
when the filter state changes.

Every `FilteredRecordArray` returned from the `store.filter` function
is referenced by the store. This prevents the garbage collector from
reclaiming the `FilteredRecordArray` after the application no longer
needs the data. This results in memory leaks in ember applications
that use `store.filter`.

For example, a common pattern is to call `store.filter` from within a
computed property.

```js
App.ExampleController = Ember.ObjectController.extend({
    filteredPosts: function() {
        var authorQuery = this.get('authorQuery');

        return this.store.filter('post', function(post) {
            return post.get('author') === authorQuery;
        });
    }.property('authorQuery')
});
```

In this example, a new `FilteredRecordArray` is created every time the
`authorQuery` property is updated. This is results in a memory leak
for tow reasons. First, the `authorQuery` variable is in a closure and
the filter function is unable to track its dependency on that
property. Second, the store doesn't know when the
`FilteredRecordArray` created by this function is no longer needed by
the application.

A second pattern that results in memory leaks is when `store.filter`
is called in a model hook.

```js
App.ExampleRoute = Ember.ObjectRoute.extend({
    model: function() {
        this.store.filter('post', {unread: true}, function(post) {
            return post.get('unread');
        });
    }
});
```

This pattern has the same problems as the above example with a
computed property. A new `FilteredRecordArray` is created every time
the route is entered resulting in a orphaned array that the garbage
collector can not reclaim.


# Detailed design


Basic Api:
-----------

Filter Objects are a new type of object available on the DS
namespace. They contain a `filterFunction` which is the function used
to filter the set of model records that the Ember Data store knows
about. Any additional state that is used by this filter function will
live as a property of this filter object. It is expected that these
properties will be an `alias` of another property located on one of
the controllers in the ember app.

The `type` property of a filter will contain the name of the model
class.

The `filterProperties` property of a filter object will be used to
explicitly track the dependent properties of the filter object.

As an optimization a `recordProperties` array can also be specified to
list the properties on the records that this filter is dependent
on. The Ember Data store can use this information to skip recomputing
the `FilteredRecordArray` if a property change will have no impact
filtered set.

An example of how to define a DS.Filter object is listed below:

```js
// app/filters/post-title.js

export default DS.Filter.extend({
  type: 'post',
  filterProperties: ['title'],
  recordProperties: ['articleTitle'], //optional
  
  filterFunction: function(record) {
    var title = this.get('title');
    return record.get('title').match(title);
  }
});
```

Route API:
-----------

The above filter object could then be used from a route by calling
`this.store.filter` and providing a string that the ember container
can resolve to an instance of the Filter Object we have defined
above. A hash can be provided as a second parameter containing the new
filter state. By default the store should always return the same
instance of a Filter Object.

For example:

```js
// app/routes/filtered-post.js
export default Ember.Route.extend({
  model: function() {
    var controller = this.controllerFor('filtered-post');
    
    return this.store.filter('post-title', {
      controller: controller,
      title: alias('controller.searchTerm')
    });
  }
});
```


Nice API Sugar:
-----------------

Use of a DS.Filter object in a controller could be made simpler by
providing a helper function which automatically creates a filter
object that aliases the controller's properties. This could be done by
having a function that takes the type, an array of dependent
property names and the filter function as arguments and returns a
`FilteredRecordArray` which is associated with a fully initialized
DS.Filter object.

For example:


```js
// app/controllers/filtered-post.js
import { filter } from "ember-data/filters";

export default Ember.Controller.extend({
  searchTerm: alias('model.title'),
  
  filteredPosts: filter('post', ['searchTerm'], function(post) {
    return post.get('title') === this.get('searchTerm')
  })
});
```
# Drawbacks

The Filter Object API is slightly complex then the current filter API.

One potential drawback is Filter Objects do not fully solve the memory
leak issue. The store will still keep a reference to all
`FilteredRecordArray` and there is no automatic mechanism for
informing the store when a `FilteredRecordArray` is no longer needed.


# Alternatives

N/A

# Unresolved questions

Ideally, we would like to have a way to tell the store when a
`FilteredRecordArray` is no longer needed. It would be even better if
Ember Data took care of this automatically and it required no action
on the user's part.
