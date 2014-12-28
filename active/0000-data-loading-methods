- Start Date: 12/22/2014
- RFC PR: 
- Ember Issue: 

# Summary

The data loading methods in Ember Data should be broken down and rearranged based on the methods by which they load and return data, whether it is from a local store, from a remote server, or a combination of both. The names of each of these methods should also be decided carefully, using the same name for both single object and collection loads, so as to avoid confusion and reduce the learning curve.

# Motivation

Currently in Ember Data the layout and naming of the data loading functions is somewhat disorganized, which increases the mental overhead of using the library. `.all` and `.getById` return local data, `.find` may look locally, except for when you pass an object (i.e. `.find({minAge: '1'})`) in which case the server is always queried. There's also a new `.fetch` function that will always request new data from the store, and possibly other methods that just aren't coming to mind at the moment.

# Detailed design

The current scheme for data loading in Ember Data is something like:

|multiplicity|local data|fetch if not local|fetch always|local, with new load in background|
|---|---|---|---|---|
|collection|store.all|?|store.find|?|
|single record|store.getById|store.find|store.fetch|?|

Where we have somewhat inconsistent naming, and an incomplete coverage of the possible loading styles. I believe a better scheme would look more like:

|multiplicity|local data|fetch if not local|fetch always|load, with new load in background|query server to use its logic|
|---|---|---|---|---|---|
|collection|store.grab|store.find|store.fetch|store.find+store.fetch|store.query|
|single object|store.grab|store.find|store.fetch|store.find+store.fetch|store.query|

There would then be 3 loading styles based on the typical needs above, as well as an extra distinction that I would like to introduce:

1. Load data *ONLY* from the local store.
2. Load data from the local store if there, otherwise fetch from the server.
3. Load data from the server, ignoring whatever is currently in the store.
4. Query the server for it's logic and load whatever models are returned.

# Load data *ONLY* from the local store.



# Load data from the local store if there, otherwise fetch from the server.



# Load data from the server, ignoring whatever is currently in the store.



# Query the server for it's logic and load whatever models are returned.



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
