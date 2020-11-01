---
Start Date: 2017-12-12
RFC PR: https://github.com/emberjs/rfcs/pull/281
Ember Issue: (leave this empty)

---

# Summary

Install ES5 getters for computed properties on object prototypes, thus
eliminating the need to use `this.get()` or `Ember.get()` to access them.

Before:

```js
import Object, { computed } from '@ember/object';

const Person = Object.extend({
  fullName: computed('firstName', 'lastName', function() {
    return `${this.get('firstName')} ${this.get('lastName')}`;
  })
});

let chancancode = Person.create({ firstName: 'Godfrey', lastName: 'Chan' });

chancancode.get('fullName'); // => 'Godfrey Chan'

chancancode.set('firstName', 'ʎǝɹɟpo⅁');

chancancode.get('fullName'); // => 'ʎǝɹɟpo⅁ Chan'

let { firstName, lastName, fullName } = chancancode.getProperties('firstName', 'lastName', 'fullName');
```

After:

```js
import Object, { computed } from "@ember/object";

const Person = Object.extend({
  fullName: computed('firstName', 'lastName', function() {
    return `${this.firstName} ${this.lastName}`;
  })
});

let chancancode = Person.create({ firstName: 'Godfrey', lastName: 'Chan' });

chancancode.fullName; // => 'Godfrey Chan'

chancancode.set('firstName', 'ʎǝɹɟpo⅁');

chancancode.fullName; // => 'ʎǝɹɟpo⅁ Chan'

let { firstName, lastName, fullName } = chancancode; // No problem!
```

# Motivation

Ember inherited its computed properties functionality from [SproutCore](http://guides.sproutcore.com/core_concepts_kvo.html).
The feature was designed at a time before [ES5 getters](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/get)
were widely available. This necessitated using a special function such as
`this.get()` or `Ember.get()` to access the values of computed properties.

Since all of our [target browsers](https://github.com/emberjs/rfcs/pull/252)
support ES5 getters now, we can drop the need of this special function,
improving developer ergonomics and interoperability between other libraries
and tooling (such as TypeScript).

Note that at present, using `this.set()` or `Ember.set()` is still mandatory
for the property to recompute properly. In the future, we might be able to
loosen this requirement, perhaps with the help of ES5 setters. However, that
would require more design and is out-of-scope for this RFC.

`this.get()` and `Ember.get()` will still work. This RFC does not propose
removing or deprecating them in the near term. They support other use cases
that ES5 getters do not, such as "safe" path chaining (`get('foo.bar.baz')`)
and `unknownProperty` (and Proxies by extension), so any future plans to
deprecate them would have to take these features into account.

Addon authors would likely need to continue using `Ember.get()` for at least
another two LTS cycles (8 releases) to support older versions of Ember (and
possibly longer to support proxies). It is, however, very unlikely that the
everyday user would need to use this.

# Detailed design

The computed property function, along with any caches, can be stored in the
object's "meta". We will then define a getter on the object's prototype to
compute the value.

One caveat is that the computed property function is currently stored on the
instances for implementation reasons that are no longer relevant. However,
it is possible that some developers have observed their existance and have
accidentally relied on these private semantics (e.g. `chancancode.fullName.get()`
or `chancancode.fullName.isDescriptor`).

Before landing this change, we should turn the property into an assertion
so that in these unlikely scenarios, developers will at least receive
some warning.

Another thing to consider is that there is this Little Known Trick™ to add
Computed Properties to POJOs:

```js
import { computed, get } from "@ember/object";

let foo = {
  bar: computed(function() { return 'bar'; })
};

get(foo, 'bar'); // => 'bar'
```

In this case, there is no opportunity for us to install an ES5 getter, and
`Ember.get` is the only solution. This is very rare in practice and is more
or less just a party trick. We should deprecate this use case (in `Ember.get`)
and suggest the alternative:

```js
import Object, { computed } from "@ember/object";

let foo = Object.extend({
  bar: computed(function() { return 'bar'; })
}).create();

foo.bar; // => 'bar'
```

Or simply...

```js
let foo = {
  get bar() {
    return 'bar';
  }
};

foo.bar; // => 'bar'
```

# How We Teach This

For the most part, this RFC _removes_ a thing that we need to teach new
users.

It might, however, come across as slightly strange that `set()` is still
required. However, many other libraries share the same model, and
empricially, this does not appear to be an issue. For example, in React,
you can freely access `this.state.foo` but must use `this.setState('foo', ...)`
to update it. Even Vue has [the same API](https://vuejs.org/v2/api/#Vue-set)
for some cases.

The mental model for this is that you must use the `set()` in order for
Ember to notice your mutations, so that it can update the caches, rerender
things on the screen, etc.

As for users who already learned to use `get()` everywhere, that would
continue to work. Ideally, this would be a Cool Trick™ they pick up some day
(as in "Oh, I don't have to do _that_ anymore? Cool."), at which point the
old habit would quickly die. If this turned out to be too confusing, we
could always explore deprecating `this.get()`; we will just have to weigh
the cost-benefits of the confusion (if any) versus churn.

# Drawbacks

As mentioned, not removing `set()` at the same time might be a source of
confusion. However, removing `set()` would require significantly more
upfront design work, and it [might not even be possible](https://vuejs.org/v2/guide/reactivity.html#Change-Detection-Caveats)
to completely remove the need of `set()` (as the system is designed today)
in all cases (see `Vue.set()`).

Since removing `get()` would unlock so many benefits, and since there are
plenty of other libraries that uses the same model, the case for decoupling
the two seems overwhemlingly positive.

# Alternatives

* Hold off until we also remove `set`
* Hold off until we transition to something like [Glimmer's `@tracked`](https://glimmerjs.com/guides/tracked-properties)

In my opinion, these alternatives do not make a lot of sense, as neither
of these hypothetical systems appear to require (or would benefit from)
having a user-land getter system.
