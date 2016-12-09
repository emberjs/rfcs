- Start Date: 2016-12-08
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Whenever a computed property updates, Ember will only continue updating the
rest of the tree if the resulting value has changed.

# Motivation

This can improve performance of Ember across the entire ecosystem without
adding additional effort in most use cases.

# Detailed design

```js

ComputedPropertyPrototype.get = function(obj, keyName) {
	let meta = metaFor(obj);
	let cache = meta.writableCache();

	let updated = false;
	for (idx = 0; idx < this._dependentKeys.length; idx++) {
		depKey = this._dependentKeys[idx];
		var propertyVersion = obj[depKey].updated(obj, keyName);
		if (cache.versions[depKey] !== propertyVersion){
			updated = true;
			break;
		}
	}

	if (!updated){
		return cache[keyName];
	}

	...
}

ComputedPropertyPrototype.updated = function(obj, keyName) {
	let previouslyUpdated = computedUpdatedMap.get(obj);
	if (previouslyUpdated !== undefined){
		return previouslyUpdated;
	}
	let meta = metaFor(obj);
	let cache = meta.writableCache();

	let previousValue = cache[keyName];
	let currentValue = Ember.get(obj, keyName);

	if (previousValue !== currentValue){
		cache.version = {}; //The version just needs to change, the value doesnt actually matter
	}

	return cache.version;
}


# How We Teach This

To teach this we should simply modify the documentation to state that computed
will update if and only if one of the dependent properties has changed.

# Drawbacks

This is a breaking change. If computeds are not used in a functional manner,
then they can potentially break with this. It also introduces a bit more
complexity into the system when you're trying to understand why a computed isnt
updating.

# Alternatives

The impact of not doing this means that users have to create rather hacky versions
of computeds to do this themselves. This will result in increased memory usage,
longer development times, and a higher cost to entry as one would have to have a
better understanding of how the compute chain works.

An alternative approach could be to either add an option to computeds of the form:

```js
{
	someProperty: Ember.computed({
		get: function(){

		},
		updateDependentsIfValueDoesntChange: false
	})
}
```

or to add a different type of computed which will not update if its dependent properties
dont update:

```js
{
	someProperty: Ember.functionalComputed({

	})
}
```

(I use the term functional computed because we can safely make the optimization of not
rerunning it if it has no side-effects. This may incur some learning costs). 

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?
