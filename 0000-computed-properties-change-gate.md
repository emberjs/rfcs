- Start Date: 2014-10-2
- RFC PR:
- Ember Issue:

# Summary

Change the way computed properties work to avoid notify observers or make other CP's depending on be fired.

# Motivation

### The Problem

I'm sure you faced this more than once. Then a CP changes from "A" to "A" (it doesn't change), any
other CP watching it is staled.


```js
var Person = Ember.Object.extend({
  initials: function(){
    console.log('Initials called!');
    return this.get('name')[0] + '.' + this.get('surname')[0];
  }.property('name', 'surname'),

  firstInitial: function(){
    console.log('First initials called!');
    return this.get('initials').split('.')[0];
  }.property('initials')
});

var obj = Person.create({name: 'John', surname: 'Doe'})

obj.get('initials');      // Calls the initials function. Returns 'J.D'
obj.get('firstInitial');  // Calls the firstInitial function. Returns 'J'

obj.set('name', 'Jane');  // Changes a value, dependencies are not invoked yet.

obj.get('initials');      // Calls the initials function. Returns 'J.D' as it did before.
console.log('Next call should not log anything, but it will');
obj.get('firstInitial');  // Calls the firstInitial function, without need. Returns 'J' again.
```

This feels wrong.
I been told that this idea is in the pipeline, but I wanted to officialize it with this RFC.

### Cavears

For avoid propagating changes when a CP is recomputed, we need to compare the old value with the new value.

This is cool for simple types, but not easy for arrays p.e.

This is totally fine, I would consider an array the same if the object is the same and its length too,
but probably many people will be confused why [1,2,3] is not equal to [1,2,3], even if its an expected
result in javascript.

If we are dealing with very big arrays and we still want to get the bennefit of being to compare arrays by its content,
it could be a good usecase for immutable data structures.

### Current workarounds

Is a well known issue.

There is even a project and screencast for doing the trick (using observers) to achieve this: https://github.com/GavinJoyce/ember-computed-change-gate
But using observers is not the idea solution. They bring synch/async duality.

# Drawbacks

- While this does not break an API, I can imagine a lot of situations where people is reliying in the current behavior, so it should
  go in a major release.

