- Start Date: 2019-04-12
- Relevant Team(s): Ember.js, Learning
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Tracked Properties Updates

## Summary

During the Ember Octane preview period we encountered some issues with the
current design for Tracked Properties that was proposed and accepted in
[RFC 410](https://github.com/emberjs/rfcs/blob/master/text/0410-tracked-properties.md).
The primary issues were specifically around interop between tracked properties,
computed properties, and autotracking, with a few extra issues and
inconsistencies surrounding these. This RFC seeks to fix these issues and
provide a new interop path.

## Motivation

During the preview for Octane, we've encountered a few issues with tracked
properties:

1. Computed property autotracking interop was too aggresive, and resulted in
   breaking changes in existing applications.
2. When users need to use `get()` and `set()` is still fairly confusing, and we
   don't have enough warnings to help guide users down the happy path.
3. Users were confused by the fact that `set()` was still required when updating
   computed properties, especially CP macros like `DS.attr()`.

### Autotracking Interop

The core of the autotracking issue was that autotracking _inside_ computed
properties resulted in more values being consumed and watched than before,
fundamentally changing the dynamic of the CP. For instance, consider this code
example:

```js
const Person = EmberObject.extend({
  init(...args) {
    this._super(...args);

    let fullName = `${this.get('firstName')} ${this.get('lastName')}`;

    this.set('fullName', fullName);
  },
});

const Profile = EmberObject.create({
  person: computed('firstName', 'lastName', function() {
    return Person.create({
      firstName: this.get('firstName'),
      lastName: this.get('lastName'),
    });
  }),
});
```

The `Profile#person` computed would currently only invalidate if
`Profile#firstName` or `Profile#lastName` was updated, and it would be possible
to update the person like so:

```js
// without autotracking
let profile = Profile.create({
  firstName: 'Chris',
  lastName: 'Thoburn',
});

let p1 = profile.get('person');
profile.set('person.firstName', 'Christopher');
let p2 = profile.get('person');

p1 === p2; // true
```

However, because `get` now autotracks, the fact that the `Person` class uses
`this.get()` in its own constructor causes `Profile#person.firstName` to
autotrack. When we go to update the object later on, it invalidates the
underlying computed property.

```js
// with autotracking
let profile = Profile.create({
  firstName: 'Chris',
  lastName: 'Thoburn',
});

let p1 = profile.get('person');
profile.set('person.firstName', 'Christopher');
let p2 = profile.get('person');

p1 === p2; // false
```

It's arguable that computed properties that are relying on these caching
semantics are problematic in general. After all, it's strange to setup state
like this _during_ construction, usually you would use a CP instead, and if CPs
_are_ trying to use a value, it generally means that value _should_ be a
dependency. However, based on our experiences with attempting to upgrade
existing applications to enable autotracking, we believe it likely would result
in enough breakage that it would be a breaking change.

### When to Use `get` and `set`

While the original tracked properties RFC laid the groundwork for getting rid of
`get` and `set` in the browser, there are still cases where users need to use
it. Specifically, users must use `get` and `set` when:

- Getting and setting values on POJOs
- Using Ember Proxies
- Setting Computed Properties

Ember proxies already throw errors if users don't use `get` and `set`, so users
can generally get the feedback they need for them, and setting computed
properties is addressed in the next section. For POJOs, however, the feedback
can be lacking. Users could access a POJO from a tracked context like so:

```js
class MyComponent extends GlimmerComponent {
  featureFlags = {};

  get someValue() {
    if (get(this.featureFlags, 'someValueEnabled')) {
      // ... do things
    }
  }
}
```

And later on, try to update `featureFlags.someValueEnabled`, and be confused
when it doesn't work and they don't have any actionable feedback:

```js
class MyComponent extends GlimmerComponent {
  featureFlags = {};

  get someValue() {
    if (get(this.featureFlags, 'someValueEnabled')) {
      // ... do things
    }
  }

  @action
  updateFlag() {
    this.featureFlags.someValueEnabled = true;
  }
}
```

Adding an assertion to values that are accessed like this which requires `set`
will help to prevent confusion from occuring.

### Computed Property Setters

As we move toward removing `get` and `set` entirely, computed properties stick
out somewhat as a sore thumb. They generally look and feel like standard getters
and setters with some caching, but while modern Ember users can use native
getters to _get_ the computed, they must use `set()` to update them. This is
particularly annoying when dealing with macros, like aliases and Ember Data
attributes such as `DS.attr`.

Installing a native setter for computed properties will smooth over these
inconistencies, and give us a clear learning boundary for `get` and `set` - you
only need to use them for plain, undecorated properties on POJOs, and for Ember
proxies.

## Detailed design

### Autotracking Interop

Not all of the interop in the original RFC was problematic, and as such, the
following parts will remain the same:

- `get` - will still autotrack any value that is accessed
- `set`/`notifyPropertyChange` - will still invalidate any value that was
  accessed with `get`
- Computed properties - will still autotrack _if accessed in an autotracking
  context_.

What will change is that computed properties will no longer autotrack as they
are being evaluated. Instead, they will follow the same rules as they do
currently, listing explicit dependencies and only invalidating when one of those
dependencies changes. Any autotracking that may have occured during the
computation of the computed will instead no-op, making the computed a _black
box_.

Since computed properties no longer autotrack, they will need a different
interop story for tracked properties and autotracking. Autotracking is a very
general tool - as we saw in the motivation, it's possible to track through
_function calls_, something that wasn't possible before.

However, we don't need to enable interop with _all_ of autotracking. All we
really need is the ability to depend on the autotracking equivalents of what
computed properties already are capable of depending on, so that users can
convert existing code to autotracking incrementally. Computed properties can
already depend on:

- Properties
- Other Computed Properties

The equivalents to these in autotracking are:

- Tracked Properties
- Native Getters

#### Depending on Tracked Properties

Tracked properties are already instrumented under the hood. They have a native
setter that calls `notifyPropertyChange`, and this will automatically invalidate
any computeds that specify the property as a dependency. There is no need to do
any further work.

#### Depending on Native Getters

Native getters are just that - native. They don't have any special autotracking
behavior, which was part of the benefits of tracked properties. However, this
means there is nothing to notify computed properties of changes.

To solve this problem, we propose the `@watchable` decorator. This decorator
would instrument a native getter with its own autotracking frame, which would
allow it to track any events in its evaluation. It would coalesce these into its
own tag, which computed properties (and observers) would be able to depend on:

```js
import { tracked, watchable } from '@glimmer/tracking';

class Person {
  @tracked firstName;
  @tracked lastName;

  @watchable
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}

const Profile = EmberObject.extend({
  // provided on create
  person: null,

  username: alias('person.fullName'),
});
```

`@watchable` would be imported from `@glimmer/tracking` alongside `@tracked`.
Like other Ember decorators, it would be usable in both classic and native
classes. When used in classic classes, it will be able to define its underlying
getter and setter using the same API as computed properties. However, it will
throw an error if it is used to define more than one getter/setter - watchable
macros should be avoided and discouraged.

Observers and computed properties will throw an error if they attempt to watch
a getter which is not marked as watchable.

#### Debugging Assertions

Consuming a value using `get()` inside of a tracked context will both autotrack
the value, and in _development_ builds install the mandatory setter assertion.
This assertion already exists and is currently installed on values that are
watched by computeds, observers, and templates, but not for values accessed
using `get()`. Extending it to these values should not be too difficult.

### Computed Property Setters

Computed properties will no longer install the mandatory setter assertion like
they have for much of Ember's existence. Instead, they will install a native
setter that proxies to the one defined for the computed property. This will
allow users to use native setters instead of `set()`.

## How we teach this

There are two major points of consideration here:

- How do we teach classic/autotrack interop and `@watchable`
- How do we teach `get`/`set` and when they are necessary to use

### Classic/Autotrack Interop

Many of the points from the original tracked property RFC remain valid, but we
will have to update the way that we teach computed properties. In some ways the
overall mental model is simplified - computed properties will only update
whenever a dependent property is updated, as they always have. The following
table describes what types of values can be depended on, and how they can
trigger updates:

| Type                        | Updates By                         |
| --------------------------- | ---------------------------------- |
| Plain, undecorator property | `set()`                            |
| Tracked property            | Native setter                      |
| Computed property           | `set()`, or upstream invalidations |
| Watchable getter            | Tracked value changes              |

We should cover each of these in some detail in the main guides.

#### `@watchable` API Docs

`@watchable` is decorator that can be used on _native getters_ that use tracked
properties. It exposes the getter to Ember's classic computed property and
observer systems, so they can watch it for changes. It can be used in both
native and classic classes.

Native Example:

```js
import { tracked, watchable } from '@glimmer/tracking';
import { computed, set } from '@ember/object';

class Person {
  @tracked firstName;
  @tracked lastName;

  @watchable
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}

class Profile {
  constructor(person) {
    set(this, 'person', person);
  }

  @computed('person.fullName')
  get helloMessage() {
    return `Hello, ${this.person.fullName}!`;
  }
}
```

Classic Example:

```js
import { tracked, watchable } from '@glimmer/tracking';
import EmberObject, { computed, observer, set } from '@ember/object';

const Person = EmberObject.extend({
  firstName: tracked(),
  lastName: tracked(),

  fullName: watchable(function() {
    return `${this.firstName} ${this.lastName}`;
  }),
});

const Profile = EmberObject.extend({
  person: null,

  helloMessage: computed('person.fullName', function() {
    return `Hello, ${this.person.fullName}!`;
  }),

  onNameUpdated: observer('person.fullName', function() {
    console.log('person name updated!');
  }),
});
```

`watchable()` can receive a getter function or an object containing `get`/`set`
methods when used in classic classes, like computed properties.

In general, only properties which you _expect_ to be watched by older, untracked
clases should be marked as watchable. The decorator is meant as an interop layer
for parts of Ember's older classic APIs, and should not be applied to every
possible getter/setter in classes. The number of watchable getters should be
_minimized_ wherever possible. New application should not need to use
`@watchable`, since it is only for interoperation with older code.

### Computed Properties

Computed properties are a pre-Octane concept in Ember. They serve the same
purpose as tracked properties and native getters, allowing users to respond to
changes, derive state, and ultimately update the DOM. They also have built-in
caching to prevent having to perform expensive calculations more than once.

While computed properties are no longer the recommended default, it's likely
that you may encounter them in code that hasn't been updated to tracked
properties just yet, either in existing applications or in the wider Ember
ecosystem, so this guide exists both to describe how they work and can be used,
and how they interoperate with tracked properties.

#### Computed Property Usage

You can create a computed property by using the `@computed` decorator to
decorate standard computed property getters and setters:

```javascript
import { computed, set } from '@ember/object';

class Person {
  constructor(firstName, lastName) {
    set(this, 'firstName', firstName);
    set(this, 'lastName', lastName);
  }

  @computed('firstName', 'lastName')
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}

let ironMan = new Person('Tony', 'Stark');

ironMan.fullName; // "Tony Stark"
```

This computed property works just like a normal getter/setter, with two key
differences:

1. It will cache its value by default, and it will only update that value if its
   _dependencies_, in this case the `firstName` and `lastName` properties,
   change.

   ```javascript
   class Counter {
     _count = 0;

     @computed
     get count() {
       console.log('counted!');
       return this._count;
     }
   }

   let counter = new Counter();

   counter.count; // logs 'counted!'
   counter.count; // logs nothing, the values was cached and hasn't updated
   ```

2. It will notify other "watchers", such as other computed properties and
   templates, if any of its dependencies has updated and it needs to be
   recalculated.

##### Specifying Dependencies

So far we've seen computed properties with dependencies on properties that are
_local_ to the object, but you can specify a few other types of dependencies:

- **Chain dependencies.** If you need to specify a dependency on an _object_,
  you can use dot notation to do so:

  ```js
  class Profile {
    constructor(user) {
      set(this, 'user', user);
    }

    @computed('user.firstName', 'user.lastName')
    get userName() {
      return `${this.user.firstName} ${this.user.lastName}`;
    }
  }
  ```

  When doing this for more than one value on the object, you can also use a
  special truncated syntax as shorthand:

  ```js
  class Profile {
    constructor(user) {
      set(this, 'user', user);
    }

    @computed('user.{firstName,lastName}')
    get userName() {
      return `${this.user.firstName} ${this.user.lastName}`;
    }
  }
  ```

  Note that no spaces are allowed in this truncated syntax, Ember will assert if
  you place any inside of it.

- **Array dependencies.** It's possible to depend on an array, and the items in
  the array, by watching the `[]` property on the array:

  ```js
  class Person {
    constructor(friends = []) {
      set(this, 'friends', friends);
    }

    @computed('friends.[]')
    get friendNames() {
      return this.friends.map(friend => friend.name);
    }
  }
  ```

  You can also depend directly on a _property_ of each item in the array using
  `@each` syntax:

  ```js
  class Person {
    constructor(friends = []) {
      set(this, 'friends', friends);
    }

    @computed('friends.@each.name')
    get friendNames() {
      return this.friends.map(friend => friend.name);
    }
  }
  ```

  However, you cannot _chain_ on these properties, as it is a performance
  pitfall. You can only do 1 level of `@each` watching.

##### Defining Setters

If you define a setter for your computed property, it'll work just like a normal
setter:

```javascript
import { computed, set } from '@ember/object';

class Person {
  constructor(firstName, lastName) {
    set(this, 'firstName', firstName);
    set(this, 'lastName', lastName);
  }

  @computed('firstName', 'lastName')
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }

  set fullName(value) {
    let [firstName, lastName] = value.split(' ');

    set(this, 'firstName', firstName);
    set(this, 'lastName', lastName);
  }
}

let hero = new Person('Tony', 'Stark');

hero.fullName; // 'Tony Stark'

hero.fullName = 'Hope Pym';
hero.firstName; // 'Hope'
```

It's worth noting that we do _not_ need to use `set` to update the computed
property. It wraps the native setter transparently, so there is no need for the
set function. The properties it _depends_ on, however, _do_ need to be updated
with `set`, since they are not marked as `@tracked` and we don't have another
way of knowing they were updated. We will dive into this a bit more below.

The setter will also immediately call the getter for the computed in order to
recalculate the cached value. You can also return the value, as an optimization:

```js
class Person {
  constructor(firstName, lastName) {
    set(this, 'firstName', firstName);
    set(this, 'lastName', lastName);
  }

  @computed('firstName', 'lastName')
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }

  set fullName(value) {
    let [firstName, lastName] = value.split(' ');

    set(this, 'firstName', firstName);
    set(this, 'lastName', lastName);

    return value;
  }
}
```

##### Computed Property Macros

It's possible to define _macros_ using computed properties. This works because
the `@computed` decorator can receive getter and setter functions, and be
applied to a normal class field instead of a getter/setter:

```js
class Person {
  constructor(firstName, lastName) {
    set(this, 'firstName', firstName);
    set(this, 'lastName', lastName);
  }

  // Just a getter function
  @computed('firstName', 'lastName', function() {
    return `${this.firstName} ${this.lastName}`;
  })
  fullName;

  // With setter and getter
  @computed('firstName', 'lastName', {
    get() {
      return `${this.firstName} ${this.lastName}`;
    },

    set(key, value) {
      let [firstName, lastName] = value.split(' ');

      set(this, 'firstName', firstName);
      set(this, 'lastName', lastName);

      return value;
    },
  })
  fullNameWithSetter;
}
```

You can then extract this decorator to create a new decorator definition:

```js
const fullNameMacro = computed('firstName', 'lastName', {
  get() {
    return `${this.firstName} ${this.lastName}`;
  },

  set(key, value) {
    let [firstName, lastName] = value.split(' ');

    set(this, 'firstName', firstName);
    set(this, 'lastName', lastName);

    return value;
  },
});

class Person {
  constructor(firstName, lastName) {
    set(this, 'firstName', firstName);
    set(this, 'lastName', lastName);
  }

  @fullNameMacro fullName;
}
```

And we can abstract this further to create a function that generates the
decorator dynamically, which allows us to reuse the macro:

```js
function fullNameMacro(firstNameKey, lastNameKey) {
  return computed(firstNameKey, lastNameKey, {
    get() {
      return `${this[firstNameKey]} ${this[lastNameKey]}`;
    },

    set(key, value) {
      let [firstName, lastName] = value.split(' ');

      set(this, firstNameKey, firstName);
      set(this, lastNameKey, lastName);

      return value;
    },
  });
}

class Person {
  constructor(firstName, lastName) {
    set(this, 'firstName', firstName);
    set(this, 'lastName', lastName);
  }

  @fullNameMacro fullName('firstName', 'lastName');
  @otherFullNameMacro fullName('first', 'last');
}
```

When you provide a getter and setter like this to `@computed`, the getter and
setter receive the `key` of the property they are decorating as the first value,
and the setter receives the actual value second. The setter also **must** return
the value to be cached - the getter will not be rerun if it does not, and the
value will be `undefined`.

##### Computed Properties in Classic Classes

Computed properties can be used in classic class syntax as well. This works by
passing the getter and setter to the `computed()` decorator just like we would
for a macro:

```js
const Person = EmberObject.extend({
  fullName: computed('firstName', 'lastName', {
    get() {
      return `${this.firstName} ${this.lastName}`;
    },

    set(key, value) {
      let [firstName, lastName] = value.split(' ');

      set(this, 'firstName', firstName);
      set(this, 'lastName', lastName);

      return value;
    },
  }),
});
```

#### Computed Property Dependency Types

You may have noticed that in the previous section, our computed properties were
depending on normal, undecorated properties. This is possible in classic Ember
if we always update those properties using Ember's `set` method, which is why
all of the examples use it. Computed properties can depend on other types of
values as well though. Altogether, the types of values are:

- Plain, undecorated object properties
- `@tracked` properties
- `@computed` properties
- `@watchable` getters
- Arrays

We'll talk about each of these individually, and discuss how they are watched
and updated.

##### Plain Properties

In all the examples above, we demonstrated computed properties that depended on
plain object properties which hadn't been otherwise decorated. This was the
default in classic Ember, before tracked properties were introduced, and it
still works today - however, to trigger updates on a plain property dependency,
you _must_ use `set`:

```js
import { computed, set } from '@ember/object';

class Person {
  constructor(firstName, lastName) {
    set(this, 'firstName', firstName);
    set(this, 'lastName', lastName);
  }

  @computed('firstName', 'lastName')
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}

let ironMan = new Person('Tony', 'Stark');

ironMan.fullName; // "Tony Stark"
ironMan.firstName = 'Anthony'; // This will throw an error
set(ironMan, 'firstName', 'Anthony'); // This will work, and update `fullName`
```

In general Ember will try to throw an error if you should use `set` to update a
value, but you didn't.

##### Tracked Properties

Computed properties can also depend directly on tracked properties, and tracked
properties do _not_ need to be updated with `set`. Updating them with normal
JavaScript update syntax will invalidate them:

```js
import { computed } from '@ember/object';
import { tracking } from '@glimmer/tracking';

class Person {
  @tracked firstName;
  @tracked lastName;

  constructor(firstName, lastName) {
    this.firstName = firstName;
    this.lastName = lastName;
  }

  @computed('firstName', 'lastName')
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}

let ironMan = new Person('Tony', 'Stark');

ironMan.fullName; // "Tony Stark"
ironMan.firstName = 'Anthony'; // Now this will work, because 'firstName' is tracked!
```

##### Computed Properties

Computed properties can depend on other computed properties. If you depend on a
computed property, it will only trigger updates if _its_ dependencies update, or
if you set it directly:

```js
import { computed } from '@ember/object';
import { tracking } from '@glimmer/tracking';

class Person {
  @tracked firstName;
  @tracked lastName;

  constructor(firstName, lastName) {
    this.firstName = firstName;
    this.lastName = lastName;
  }

  @computed('firstName', 'lastName')
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }

  set fullName(value) {
    let [firstName, lastName] = value.split(' ');

    this.firstName = firstName;
    this.lastName = lastName;
  }

  @computed('fullName')
  get legalName() {
    return this.fullName;
  }
}

let hero = new Person('Tony', 'Stark');

hero.legalName; // 'Tony Stark'

hero.fullName = 'Hope Pym'; // Invalidates `legalName`
hero.legalName; // 'Hope Pym'

hero.firstName = 'Hank'; // Invalidates `fullName` _and_ `legalName`
hero.fullName; // 'Hank Pym'
hero.legalName; // 'Hank Pym'
```

##### Watchable Getters

In modern, fully tracked classes, computed properties aren't recommended
anymore. However, if you are working in a legacy codebase and converting to
tracked properties and native getters, there may be a point in time where you
try to convert a computed property that is being depended on by _other_ computed
properties. Native getters normally _cannot_ be depended on, and this will
trigger an error in development mode.

However, this doesn't mean that you need to convert an entire tree of computed
properties every time you try to update a class! Instead, you can mark native
getters that need to be watched by computed properties with the `@watchable`
decorator:

```js
import { computed } from '@ember/object';
import { tracking, watchable } from '@glimmer/tracking';

class Person {
  @tracked firstName;
  @tracked lastName;

  constructor(firstName, lastName) {
    this.firstName = firstName;
    this.lastName = lastName;
  }

  @watchable
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }

  set fullName(value) {
    let [firstName, lastName] = value.split(' ');

    this.firstName = firstName;
    this.lastName = lastName;
  }

  @computed('fullName')
  get legalName() {
    return this.fullName;
  }
}
```

This decorator exposes the getter to computed properties, but otherwise leaves
it untouched - it'll operate just like a normal native getter with tracked
properties. When you have removed all computed properties that are watching the
getter, you can remove the `@watchable` decorator.

In general, you should try to remove `@watchable` decorators as you convert your
app. Making getters watchable means that more computeds can be written to watch
those getters, and the situation can get _worse_ instead of better over time. If
you need to write a service or class that needs to interop with modern and
classic code for some time, try to _minimize_ the number of `@watchable` getters
to just the ones that are the "public API" of the class - the values that are
expected to be watched from the outside by other classes.

##### Arrays

As we mentioned above, computed properties can specify dependencies on arrays.
They can watch for changes in the items of the array by watching the `[]` key of
the array, and they can watch for changes on properties of the items using the
`@each` syntax.

In order to be properly notified of changes to an array, you either use KVO
compliant methods of Ember arrays such as `pushObject` or `popObject`, or `set`
the entire array:

```js
import { computed, set } from '@ember/object';
import { A as emberA } from '@ember/array';

class Person {
  constructor(friends = []) {
    set(this, 'friends', friends);
  }

  @computed('friends.[]')
  get friendNames() {
    return this.friends.map(friend => friend.name);
  }
}

let joey = new Person(
  emberA([
    { name: 'Phoebe' },
    { name: 'Monica' },
    { name: 'Chandler' },
    { name: 'Ross' },
  ])
);

// Using pushObject will cause `friendNames` to update
joey.friends.pushObject({ name: 'Rachel' });

// Alternatively, we can update the whole array:
set(joey, 'friends', [...joey.friends, { name: 'Rachel' }]);
```

If the property is tracked, then `set` is not necessary, and the field can be
updated directly as you would with normal tracked properties:

```js
import { computed } from '@ember/object';
import { tracked } from '@glimmer/tracking';

class Person {
  @tracking friends;

  constructor(friends = []) {
    this.friends = friends;
  }

  @computed('friends.[]')
  get friendNames() {
    return this.friends.map(friend => friend.name);
  }
}

let joey = new Person([
  { name: 'Phoebe' },
  { name: 'Monica' },
  { name: 'Chandler' },
  { name: 'Ross' },
]);

joey.friends = [...joey.friends, { name: 'Rachel' }];
```

#### Computed Properties and Tracking

Computed properties will autotrack when they are accessed from templates or
through other getters, like tracked properties.:

```js
import { computed } from '@ember/object';
import { tracking, watchable } from '@glimmer/tracking';

class Person {
  @tracked firstName;
  @tracked lastName;

  constructor(firstName, lastName) {
    this.firstName = firstName;
    this.lastName = lastName;
  }

  @computed('firstName', 'lastName')
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }

  set fullName(value) {
    let [firstName, lastName] = value.split(' ');

    this.firstName = firstName;
    this.lastName = lastName;
  }

  // legalName will update whenever `fullName` updates
  get legalName() {
    return this.fullName;
  }
}
```

### When to Use `get` and `set`

Ember's classic change tracking system used two methods to ensure that all data
was accessed properly and updated correctly: `get` and `set`.

```js
import { get, set } from '@ember/object';

let person = {};

set(person, 'firstName', 'Amy');
set(person, 'lastName', 'Lam');

get(person, 'firstName'); // 'Amy'
get(person, 'lastName'); // 'Lam'
```

In classic Ember, all property access had to go through these two methods. Over
time, these rules have become less strict, and now they have been minimized to
just a few cases. In general, in a modern Ember app, you shouldn't need to use
them all that much. As long as you are marking your properties as `@tracked`,
autotracking should automatically figure out what needs to change, and when.

However, there still are two cases where you _will_ need to use them:

- When accessing and updating plain, undecorated properties on objects
- When using Ember's `ObjectProxy` class

Additionally, you will have to continue using _accessor_ functions for arrays if
you want arrays to update as expected. These functions are covered in more
detail in the guide on arrays (LINK TO ARRAY GUIDES HERE).

Importantly, you do _not_ have to use `get` or `set` when reading or updating
computed properties, as was noted in the computed property section.

#### Plain Properties

In general, if a value in your application could update, and that update should
trigger rerenders, then you should mark that value as `@tracked`. This
oftentimes may mean taking a POJO and turning it into a class, but this is
usually better because it forces us to _rationalize_ the object - think about
what its API is, what values it has, what data it represents, and define that in
a single place.

However, there are times when data is _too_ dynamic. As noted below, proxies are
often used for this type of data, but usually they're overkill. Most of the
time, all we want is a POJO.

In those cases, you can still use `get` and `set` to read and update state from
POJOs within your getters, and these will track automatically and trigger
updates.

```js
class Profile {
  person = {
    firstName: 'Chris',
    lastName: 'Thoburn',
  };

  get profileName() {
    return `${get(this.person, 'firstName')} ${get(this.person, 'lastName')}`;
  }
}

let profile = new Profile();

// render the page...

set(profile.person, 'firstName', 'Christopher'); // triggers an update
```

This is also useful for interoperating with older Ember code which has not yet
been updated to tracked properties. If you're unsure, you can use `get` and
`set` to be safe.

#### `ObjectProxy`

Ember has and continues to support an implementation of a [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy),
which is a type of object that can _wrap around_ other objects and _intercept_
all of your gets and sets to them. Native JavaScript proxies allow you to do
this without any special methods or syntax, but unfortunately they are not
available in IE11. Since many Ember users must still support IE11, Ember's
`ObjectProxy` class allows us to accomplish something similar.

The use cases for proxies are generally cases where some data is very dynamic,
and its not possible to know ahead of time how to create a class that is
decorated. For instance, [ember-m3](https://github.com/hjdivad/ember-m3) is an
addon that allows Ember Data to work with dynamically generated models instead
of models defined using `@attr`, `@hasMany`, and `@belongsTo`. This cuts back on
code shipped to the browser, but it means that the models have to _dynamically_
watch and update values. A proxy allows all accesses and updates to be
intercepted, so M3 can do what it needs to do without predefined classes.

Most `ObjectProxy` classes have their own `get` and `set` method on them, like
`EmberObject` classes. This means you can use them directly on the class
instance:

```js
proxy.get('firstName');
proxy.set('firstName', 'Amy');
```

If you're unsure whether or not a given object will be a proxy or not, you can
still use Ember's `get` and `set` functions:

```js
get(maybeProxy, 'firstName');
set(maybeProxy, 'firstName', 'Amy');
```

## Drawbacks

> Why should we _not_ do this? Please consider the impact on teaching Ember,
> on the integration of this feature with other existing and planned features,
> on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
> TBD?
