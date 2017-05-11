- Start Date: 2015-02-04
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Ember Data wraps typical CRUD actions nicely. However, it is lacking when it comes to endpoints that mutate the state of the world without expecting typical request bodies. We could develop an API that makes interacting with those endpoints easier.

# Motivation

To expose an API to interact with non-CRUD endpoints.

# Detailed design

Imagine that we have a commenting API.

Administrators can curate comments, marking spammy or offensive records as spam.

* `GET /comments` exposes the collection
* `GET /comments/:id` exposes a particular comment
* `POST /comments/:id/spam` marks a comment as spam

What is the best way to interact with the `/comments/:id/spam` endpoint?

http://emberjs.jsbin.com/befonozine/1/edit

```javascript
App.Comment = DS.Model.extend({
  name: DS.attr("string"),
  body: DS.attr("string"),
  
  reportSpam: function() {
    var adapter = this.store.adapterFor("comment");
    var url = adapter.buildURL("comment", this.get("id"), this);
    
    return adapter.ajax(url + "/spam", "post");
  }
});

App.CommentAdapter = App.ApplicationAdapter.extend({
  reportSpam: function(comment) {
    var url = this.buildURL("comment", comment.get("id"), this);
    
    return this.ajax(url + "/spam", "post");
  }
});

App.IndexRoute = Ember.Route.extend({
  model: function() {
    return this.store.find("comment");
  },
  
  actions: {
    reportSpam: function(comment) {
      // first choice
      comment.reportSpam().then(function() {
        // do something
      });
      
      // second choice
      this.store.adapterFor("comment").reportSpam(comment).then(function() {
        // do something
      });
    }
  }
});
```


This JS Bin explores some possible approaches.

Neither are particularly pretty, and both might even depend on private / internal methods.

# Drawbacks

If you argue that `spam` should be a `DS.attr("boolean")` on the comment, and toggling that flag should be handled on the server, this RFC seems like a bad idea.

From an API design standpoint, exposing an additional, focussed endpoint is easier and cleaner than monitoring the state and mutation of a flag on a record, buried in an otherwise conventional controller action.

# Alternatives

If we don't address this, interacting with non-CRUD endpoints could lead to heinous workarounds on the front-end, and/or heinous workarounds on the backend.

# Unresolved questions

What could we expose to make this easier?

Should the model itself know about its URL?

Should the store know how to translate special model functions to special API endpoints? Should the adapter? 
