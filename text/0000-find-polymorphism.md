- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Add a `{ polymorphic: true }` flag to `store.findAll`, `store.findRecord`, `store.peekAll`,
`store/peekRecord` which would include subclasses in the results.

# Motivation
# Current State
Model Polymorphism is a popular and widely used feature of Ember Data. It is also supported
out of the box in JSONApi.
 Polymorphism is a natural way to model various domains, such as having `Comment` superclass
and `TextComment`, `VideoComment` subclasses, or a `Shape` superclass that has `Line` and `Circle` as subclasses.

A more advanced use case which is supported today is using Mixins for polymorphism.
Consider a Social Network app, where several different object can be commented on.

```js
var Commentable = Ember.Mixin.create({
  comments: hasMany('comment')`
});
```

```js
var BlogPost = DS.Model.extend(Commentable, {
  title: DS.attr();
});
```

```js
var Comment = DS.Model.extend({
  parent: DS.belongsTo('commentable', { polymorphic: true });
});
```

The above design allows Ember Data to keep parent/comments OneToMany relationship in sync, which
would not be possible without being explicit about the polymorphic mixin.

However, today Ember Data only supports polymorphism in relationships. If for example you are building
an online mocking tool, your models might look like:

```js
var Shape = DS.Model.extend({
  color: DS.attr();
});
```

```js
var Circle = Shape.extend({
  radius: DS.attr();
});
```

```js
var Line = Shape.extend({
  length: DS.attr();
});
```

```js
var Canvas = DS.Model.extend({
  shapes: DS.hasMany('shape', { polymorphic: true });
});
```

Currently you can do `canvas.get('shapes')` and get Lines and Circles, but you cannot do `store.findAll('shape')`
and get all the Lines and Circles.

# Detailed design

## FindAll and PeekAll
`store.findAll('shape', { polymorphic: true })` would return a live array that would have all the shapes and all of the
shape subclasses.

We would implement a private method, `store._subclassesFor`:
```
  /*
    Returns all the Model types that are subclasses of `type`
  */
  store._subclassesFor(type)
```

`store.findAll('shape', { polymorphic: true })` would listen to live arrays of 'shape' and of all the classes and concat them.
`store.peekAll('shape', { polymorphic: true })` would behave in the same way.

## FindRecord and PeekRecord

### Fetching case
In current Ember Data, requesting `store.findRecord('shape', 1, { reload: true })` and getting back `{ data: { id:1, type: 'line' } }` is an error
condition.

`store.findRecord('shape', id, { reload:true,  polymorphic: true })` and getting back `{ data: { id:1, type: 'line' }}` would not error out, but
rather check if there exists a different shape with `id:1` in the identity map, and if it does Error out. If it does not, it would
return the Line DS.Model from the findRecord promise.

### Caching case
`store.findRecord('shape', id)` currently checks the identity map for existence of the shape record with `id:1` with `type:'shape'`
`store.findRecord('shape', id, { polymorphic: true })` would also check the identity map of it's subclasses.
'
## URLs
We would make no assumptions about URL structure with regards to polymorphism. In particular `line.save()` would go to `/lines` by default.
It is very easy for the user to override this and in the LineAdapter.

# Drawbacks
Added complexity.
Having to pass `{ polymorphic: true }` to each invokation of find is repetative and noisy, and might be easy to forget. However, if you forget,
you will immediately see that you are not getting data, and debug. (compare with alternatives, where bugs occur later).
Composition over Inheritance
This approach is slightly refactor hostile. If you start with a Model structure that only has shapes and later break them up
into subclasses you have to track down every place you used `find` and pass `{ polymorphic: true }`. 

# Alternatives
Kris proposed that we should have an `abstractBaseClass` property on models.
The above example would be rewritten as:

```js
var Shape = DS.Model.extend({
  color: DS.attr();
});
Shape.reopenClass({
  abstract: true
});
```

```js
var Circle = Shape.extend({
  radius: DS.attr();
});
```

```js
var Line = Shape.extend({
  length: DS.attr();
});
```

This approach has several advantages:
`store.findAll('shape')` would do the right thing without having to be passed `{ polymorphic: true }`
It would be much easier for `line.save()` to automagically go to /shapes

However, I believe that the drawbacks are more significant:

  1.This approach is much more restrictive, you can either always include or not include subclasses,
    but cannot decide at invocation time if you want to opt out
  2. In larger teams it might be surprising that one day `store.findAll('comments')` is returning other
random models just because somebody on the team added an abstractBaseClass: true to the `comment` model.
This is a bigger problem than forgetting to add `{ polymorphic: true }` because errors can happen in a completely
different part of the app.
  3. This approach is incosistent with our current relationship polymorphism. It would require us to either deprecate
`hasMany({ polymorphic: true })` (and we just released 1.0 :( ) or to use an inconsistent mixed approach.


## Auto discovery of heirarchy

The above would be much improved with auto discovery of heirarchy. This would remove the need for `abstractBaseClass` and the `polymorphic` flag on `find` etc could be inferred from whether a class has any descendants.

```
\\ auto discovery that works in ED 2.8.0
import Ember from 'ember';

const { Service, computed, getOwner, get } = Ember;

export default Service.extend({
	_derivedTypes: null,
	_ancestorTypes: null,

	_dataAdapter: computed({
		get () {
			return getOwner(this).lookup('data-adapter:main');
		}
	}),

	_buildModelHierarchy () {
		const modelTypes = get(this, '_dataAdapter').getModelTypes();
		const modelNames = modelTypes.mapBy('name');
		const modelClasses = modelTypes.mapBy('klass.superclass'); // not sure why but apparently what we're given is derived from the model classes, rather than being the actual model class
		const derivedTypes = {};
		const ancestorTypes = {};

		for (let i = 0; i < modelNames.length; i++) {
			const baseModelClass = modelClasses[i];
			const baseModelName = modelNames[i];

			if (!derivedTypes[baseModelName]) {
				derivedTypes[baseModelName] = [];
			}

			for (let j = 0; j < modelNames.length; j++) {
				const derivedModelName = modelNames[j];
				if (!ancestorTypes[derivedModelName]) {
					ancestorTypes[derivedModelName] = [];
				}

				if (i !== j && baseModelClass.detect(modelClasses[j])) {
					derivedTypes[baseModelName].push(derivedModelName);
					ancestorTypes[derivedModelName].push(baseModelName);
				}
			}
		}

		this.setProperties({
			_derivedTypes: derivedTypes,
			_ancestorTypes: ancestorTypes
		});
	},

	derivedTypes: computed({
		get () {
			if (!get(this, '_derivedTypes')) {
				this._buildModelHierarchy();
			}

			return get(this, '_derivedTypes');
		}
	}),

	ancestorTypes: computed({
		get () {
			if (!get(this, '_ancestorTypes')) {
				this._buildModelHierarchy();
			}

			return get(this, '_ancestorTypes');
		}
	}),

	getDerivedTypes (baseTypeName) {
		const derivedTypes = get(this, 'derivedTypes');
		if (!derivedTypes) {
			return null;
		}
		return baseTypeName ? derivedTypes[baseTypeName] : derivedTypes;
	},

	getAncestorTypes (typeName) {
		const ancestorTypes = get(this, 'ancestorTypes');
		if (!ancestorTypes) {
			return null;
		}
		return typeName ? ancestorTypes[typeName] : ancestorTypes;
	}
});
```

# Unresolved questions

