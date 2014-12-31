- Start Date: 12/22/2014
- RFC PR: 
- Ember Issue: 

# Summary

I believe that the data loading methods in Ember Data should be broken down and rearranged based on the methods by which they load and return data, whether it is from a local store, from a remote server, or a combination of both. The names of each of these methods should also be carefully considered (likely using the same name for both single object and collection loads) so as to avoid confusion and reduce any learning curves.

# Motivation

Currently in Ember Data the layout and naming of the data loading functions is somewhat disorganized, which increases the mental overhead of using the library. `.all` and `.getById` return local data, `.find` may look locally or remote, except for when you pass an object (i.e. `.find({minAge: '1'})`) in which case the server is always queried. There's also a new `.fetch` function that will always request new data from the store, and possibly other methods that just aren't coming to mind at the moment.

# Detailed design

The current scheme for data loading in Ember Data is something like:

|multiplicity|local data|fetch if not local|fetch always|local, with new load in background|
|---|---|---|---|---|
|collection|store.all|?|store.find (query)|?|
|single record|store.getById|store.find|store.fetch|?|

(borrowed from [here](https://gist.github.com/igorT/577a5f27c9bc5a6bf59c#file-ed-md))

... where we have somewhat inconsistent naming, and an incomplete coverage of the possible loading styles. I believe a better scheme would look more like:

|multiplicity|local data|fetch if not local|fetch always|load, with new load in background|query server to use its logic|
|---|---|---|---|---|---|
|collection|store.grab|store.find|store.fetch|store.find+store.fetch|store.query|
|single object|store.grab|store.find|store.fetch|store.find+store.fetch|store.query|

There would then be 3 loading styles based on the typical needs above, as well as an extra distinction that I would suggest making:

1. Load data *ONLY* from the local store.
2. Load data from the local store if there, otherwise fetch from the server.
3. Load data from the server, even if it's already in the local store.
4. Query the server for it's logic and load whatever models are returned.

# Load data *ONLY* from the local store.

This function (name could be something like `.grab`) would load data only from the local store. It could be used by passing in either a single ID or a list of IDs. This function would be useful to see what data is already loaded and query the local state of the application.

# Load data from the local store if there, otherwise fetch from the server.

This function (`.find` would be the most fitting name) would be responsible for looking for data in the local store, and if it cannot find that data it would request it from the server. It could be used by passing in a type in addition to either a single ID or a list of IDs. This would be the main data loading method for most applications, and it should always try to do the minimum amount of work. If an array is passed in it would only query the server for the objects which cannot be found in the local store. As an extension: if a hash is passed (i.e. the user wants to find all objects with matching properties) it could look to see whether a matching query was previously made, and return the matches from the local store instead of querying the server.

# Load data from the server, even if it's already in the local store.

This is essentially an extension of what `.fetch()` currently does. The method would take in either an ID or a list of IDs, and return the object(s) loaded freshly from the server. This is useful when you aren't sure whether you have an object loaded locally but want to force a new version to load either way. Similar behaviour can be achieved using other methods, but this function is useful for convenience.

# Query the server for it's logic and load whatever models are returned.

This method would have behaviour similar to the current find method when a hash is passed i.e.  `.find({})`. This differs from any sort of object based `.find()` call as this relies on the servers logic to determine which records match the request (keys in the hash need not match up with properties on the model). This sort of helper can be useful when there are complexities in your domain model that you want to keep on the server and not duplicate on the client.

# Drawbacks & Alternatives & Unresolved questions

I think that no matter what, the data loading methods of Ember Data should be cleaned up before the library hits 1.0. The only question in my mind is what the best approach to this would be. There are of course many possible solutions and each is going to solve a different problem and be more optimal for different sets of users. Another way to better define data loading in Ember Data would be to use more of a hash/options based system, where there is a single entry point and the user can pass in options/queries, and the library does it's best to fulfill them.

```javascript
this.get('store').load('comment', {
  method: 'local+remote',
  query: [1,2,3],
});

this.get('store').load('post', {
  method: 'local',
  query: {author: 'jonathan'},
});
```
