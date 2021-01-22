- Start Date: 2019-12-08
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: https://github.com/emberjs/rfcs/pull/560
- Tracking: (leave this empty)

# Adding Equality Operators to Templates

## Summary

Add new built-in template `{{eq}}` and `{{neq}}` helpers to perform basic equality operations in templates, similar to those included in `ember-truth-helpers`.

This RFC is a subset of the changes proposed in #388.

## Motivation

It is a very common need in any sufficiently complex Ember app to perform some equality operations and often the most convenient place to do it is right in the templates.
Because of that, [ember-truth-helpers](https://github.com/jmurphyau/ember-truth-helpers) is one of the most installed addons out there, either directly by apps or indirectly by
other addons that those apps consume.

The fact that `ember-truth-helpers` is so popular is a good signal that this it is filling a perceived gap in Ember's functionality.

A second reason is that it might help make Ember more approachable to newcomers that have some experience in other frameworks.
Most if not all web frameworks have some way of comparing values in the templates and it's surprising that Ember requires an third party package to perform
even the most basic operations.


## Detailed design

Add `{{eq}}` and `{{neq}}` helpers.

#### `{{eq}}`
Binary operation. Throws an error if not called with exactly two arguments.
Equivalent of <arg1> === <arg2>
This is identical to the `eq` helper in `ember-truth-helpers`

#### `{{neq}}`
Binary operation. Throws an error if not called with exactly two arguments.
Equivalent of <arg1> !== <arg2>
This is identical to the `not-eq` helper in `ember-truth-helpers`, except for the name.

This RFC intentionally leaves the implementation details unspecified, those could be implemented in Glimmer VM or
in a higher level in Ember itself.

## How we teach this

While the introduction of these helpers doesn't introduce new concepts, as helpers like these could be
written and in fact were written for a long time, it might affect slightly how we frame some concepts in the guides.

Previously users were encouraged to put computed properties in the javascript file of the components, even for
the most simple tasks like negating a comparing two values using `computed.eq`.

With the addition of these helpers users don't have to resort to computed properties for simple operations, which sometimes
forced users to create javascript files for what could have been template-only components.

In addition to documenting the new helpers in the API docs, the Guides should be updated to favour the usage of helpers
over computed properties where it makes more sense, adding illustrative examples and stressing out where
the definition of truthiness of handlebars differs from the one of Javascript.

### Note on Object Equality

We should also add an additional section to the guides or API docs which discusses using object equality in templates.
In general, object equality in JavaScript can be tricky. There are times when it makes perfect sense, for instance finding
out if an item is the currently selected item in a list:

```js
class MySelect extends Component {
  items = [{ value: 1 }, { value: 2 }, { value: 3 }];

  @tracked selectedItem = this.items[0];

  isSelected(item) {
    item === this.selectedItem;
  }
}
```

The `{{eq}}` helper can be used in a similar way in templates:

```hbs
<select>
  {{#each this.items as |item|}}
    <option selected={{eq item this.selectedItem}}>
      {{item.value}}
    </option>
  {{/each}}
</select>
```

There are many valid use cases for object equality. However, there are also times when object equality is not guaranteed,
especially in cases where it would have been in Classic Ember. Consider this component:

```js
class MyComponent extends Component {
  @computed('foo')
  get someObj() {
    return { foo: this.foo }
  }

  checkEqual() {
    return this.someObj === this.someObj;
  }
}
```

`checkEqual` will return `true`, because `@computed` _caches_ the object itself, and returns the same object every time unless `foo`
changes. With Ember Octane, though, by default getters are not cached:

```js
class MyComponent extends Component {
  get someObj() {
    return { foo: this.foo }
  }

  checkEqual() {
    return this.someObj === this.someObj;
  }
}
```

Now, the `someObj` getter will rerun every time the property is accessed, returning a _new_ object every time. `checkEqual` will
now always return `false`, since the objects are not equal to each other.

Now, we can do the same thing in a template with `eq`:

```js
{{eq this.someObj this.someObj}}
```

And the result depends here on Ember template's caching strategy. Ember only accesses a given property _once_, and then it caches
the result, so this will return `true`. However, the semantics of template caches are not guaranteed, and in time may change, so relying
on object equality in this way is not generally a good pattern.

Even if the semantics do not change, there are still observable ways that users can trigger the getter twice and generate another object.
For instance:

```js
class MyComponent extends Component {
  get someObj() {
    return { foo: this.foo }
  }

  get someObjAlias() {
    return this.someObj;
  }
}
```

```hbs
{{eq this.someObj this.someObjAlias}}
```

Overall, the point here is that if users expect an object generated by a getter or helper to remain _stable_ between accesses, such that
object equality or state can be valid, then the user should explicitly cache that value themselves. This can be accomplished in a number
of ways, one option being the proposed `@cached` decorator:

```js
class MyComponent extends Component {
  @cached
  get someObj() {
    return { foo: this.foo }
  }
}
```

## Drawbacks

Adding new helpers increases the surface area of the framework and the code the core team commits to support long term.

## Alternatives

One alternative path is don't do anything and let users continue to define their own helpers (or install `ember-truth-helpers`).

## Unresolved questions

- If an app already use `ember-truth-helpers`, the `{{eq}}` helper will conflict with the one proposed here. How do we
  update `ember-truth-helpers` to make sure the helper of the same name doesn't collide with the built-in one?
- The inequality helper proposed in this RFC is `{{neq}}` while the one in ember-truth-helpers is `{{not-eq}}`. It is
  worth considering the benefits that keeping the same name might have in helping apps and addon migrate to the built-in helper.
