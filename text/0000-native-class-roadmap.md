- Start Date: 2018-06-14
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Native Class Roadmap

## Summary

Now that we’ve (almost) achieved feature parity for using native classes to
extend from EmberObject, the next step for Ember is to move toward using and
recommending native classes by default, with the long term goal of deprecating
EmberObject and removing it from the framework. This RFC seeks to outline where
we want to go (e.g. what Ember looks like without EmberObject), constraints we
face, and potential paths forward.

## Design Constraints

In order to figure out the path forward to native classes, we need to first
develop an idea of what the future of Ember looks like once we have fully
adopted them and removed EmberObject entirely. Below are several design
considerations for the new object model:

- **It should leverage the platform.** We should use and encourage standard
Javascript classes and idioms wherever possible. This means leveraging classes,
class fields, and decorators for most features that the object model provides.
- **Ember should remain as neutral as possible about the implementation details.**
Part of the reason we are going to have migration pains as a community moving
forward is that Ember is tied to the behavior of EmberObject in a myriad of
small ways. In an ideal world, these details would be opaque to Ember, allowing
users to implement and change-out object models at their own convenience. This
would have made the transition much easier, and will provide flexibility in the
community for pursuing new language features and libraries (e.g. mixins)
- **It should be possible to adopt incrementally.** This will be one of the largest
changes the community has ever gone through, on the same level if not larger than
dropping Views and adopting Components. EmberObject is pervasive - it's been
used for every construct imaginable in Ember apps and addons due to its
versatility and simplicity. Ideally users will be able to make incremental
changes toward pure native classes by first rewriting their existing classes
using native class syntax (as is already possible) and then dropping EmberObject
altogether.
- **We should consider the long tail of adoption, particularly for addons.** There
will be a long period where the community will need to support older versions of
Ember at the same time as newer versions. Ideally it should be possible to
support both the new and old model at once without separate code paths, or the
overlap period should be long enough that addons can drop EmberObject without
breaking Ember versions that are still part of their support matrix.
- **If possible, conversion should be automate-able.** This is not a hard
constraint, since it limits us to having very similar behavior to the existing
object model. However, if it is possible to create scripts that speed up
adoption it will help the community tremendously, so if we are at an impasse on
particular behaviors or implementations, we should consider whether one would be
easier to automate towards.

## Detailed design

As mentioned above, one of the main reasons this conversion is going to be a
difficult and long process is the fact that both user code and Ember code are
very intertwined with the existing object model. Specifically:

- They rely on the behavior of static methods such as `create`, `extend`, and
  `destroy`.
- They rely on the behavior of EmberObject's constructor
- They rely on a non-trivial amount of functionality provided via inheritance
- They rely on features of EmberObject's inheritance model that do not exist in
native Javascript (Mixins, concatenated/merged properties, nuances of observers
and listeners, etc.)

The fact that these are so intertwined makes it very difficult (if not
impossible) for us to rewrite the core classes that exist in Ember without
breaking user land code.

One strategy we can implement to prevent this from happening again in the future
is to make the details of classes opaque to Ember. Rather than providing base
classes that can be extended by users to add functionality and implement their
applications, we can rewrite Ember to treat user classes as delegates. In other
words, Ember will make no assumptions about the behavior of a class, other than
that it fulfills a certain interface.

For instance, a Component can be any class which receives an `args` hash and
optionally implements any of the component lifecycle hooks:

```js
export default class FooComponent {
  didInsertElement() {
    if (this.args.isShowing) {
      // do something
    }
  }
}
```

This method allows users to provide base-less classes, classes which do not
extend anything, to Ember as well. Ember could continue to provide empty base
classes to extend from, but it wouldn't have to.

Following a strategy of delegates and interfaces would disentangle Ember from
the details of the object model as much as possible. Ember would not care about
the implementation of a class, it would simply assume that it operates by the
same rules as standard Javascript classes do.

This method would also allow us to extract CoreObject from Ember and provide it
as a separate package for users who want to continue using the legacy object
model. So long as their objects match the new interfaces provided by Ember, it
shouldn't matter what class system they are using.

> Note: Just because we could continue to provide CoreObject as a separate
package doesn't mean we _should_. The point is that this strategy makes us
flexible enough to support it, if we choose to.

### Implementation and Adoption

The largest concerns for adopting the delegate pattern are:

1. Implementing the delegate pattern before removing EmberObject, so app
developers can switch seamlessly.
2. Giving a large enough buffer to addon authors so that they can switch to the
delegate pattern without dropping support for Ember versions that are still in
common use.

We can solve #1 during the Ember v3 lifecycle by changing Ember's code
internally to treat the existing base classes and native classes as delegates.
However, we cannot count on being able to remove EmberObject for v4, since
it's likely that there will still be users of Ember versions that do not support
delegates. So, the overall roadmap would look like:

- Ember v3: Implement delegates in Ember, allowing users to adopt the new world.
- Ember v4: Deprecate EmberObject, and prepare the framework for removal.
- Ember v5: Remove EmberObject.

## How we teach this

Once implemented, we can begin providing examples of creating base-less classes
for all of Ember's standard constructs. We should update the guides to recommend
this method, make announcments, and move toward officially deprecating
EmberObject.

We should also write migration guides that help users through the process of
converting to native classes. In particular, non-Ember constructs that use
EmberObject will likely have to change a fair amount. Demonstrating various
conversions in blog posts and guides will help to show people the path forward,
and spread knowledge of best practices when using native classes.

## Drawbacks

This approach limits what we can do significantly in framework code (no
inheritance, can't provide defaults, must provide all behavior external to user
code). There are many established software patterns other than inheritance for
providing most behavior, so in practice this shouldn't be an issue.

## Alternatives

- **Gradually rewrite current base classes.** We could attempt to gradually rewrite
Ember constructs such as Services, Routes, and Components in place. This would
give us more flexibility for a lot of things, like constructor behavior and
default method implementations, etc. It would probably take about as long to do
this as the delegate pattern, and could get very messy along the way, especially
for removing certain constructs (like Mixins).
- **Provide new base classes at the same time as the old ones.** We could provide
new base classes, exposed at different import paths, that users could gradually
switch to over time. This would give us more flexibility as well, but it would
also be about as much work and the same adoption timeline as the delegate
pattern, since Ember would have to support both the old Ember object model and
the new base classes at the same time until the switch. It would also likely be
fairly confusing to users, since they would have to make sure they were
importing the correct classes.

## Unresolved questions

If we go this direction, it means we cannot specify any base class behavior. One
issue that arises is the fact that dependency injections will not be available
to the class _during_ object construction. They can be assigned _after_ only.

There are a few options for how we could resolve this:

1. **Provide injected properties as named params in the constructor.** This would
allow us to mirror the injection definitions, which is fairly intuitive.
However, accessors and methods which depend on services will not work unless
the user assigns the service, which may be counterintuitive:

  ```js
  class Foo {
    @service time;

    getCurrentTime() {
      return this.time.now();
    }

    constructor({ time }) {
      this.currentTime = time.now(); // this works
      this.currentTime = this.getCurrentTime(); // this does not
    }
  }
  ```

2. **Provide injected properties as positional params to the constructor, using a
separate decorator.** This would completely separate named injections from
constructor injections entirely. This would likely be confusing to newcomers,
but would be similar to how other DI systems work:

  ```js
  @inject('time', 'date')
  class Foo {
    @service time;

    constructor(time, date) {
      this.currentTime = time.now();
      this.currentDate = date.today();
    }
  }
  ```


3. **Utilize private global state to provide the container to injections during
construction.** If done correctly, this would allow users to use injected
properties during all phases of construction. This would be the most intuitive
for users since it mirrors the existing behavior, but would be the most
difficult to maintain:

  ```js
  const constructionContainers = [];

  function getOwner(obj) {
    return OWNER in obj
      ? obj[OWNER]
      : constructionContainers[constructionContainers.length - 1];
  }

  function service(target, key, desc) {
    desc.get = function() {
      return getOwner(this).lookup(`service:${key}`);
    }
  }

  class Container {
    lookup(className) {
      let Class = this.registry.lookup(className);

      // We push the current container onto a stack, and pop it off when done
      // constructing, because other lookups may occur during the construction
      // of this object
      constructionContainers.push(this);
      let instance;

      try {
        instance = new Class();
        instance[OWNER] = this;
      } finally {
        // remove the current container, even in the case of an error
        constructionContainers.pop();
      }

      return instance;
    }
  }

  class Foo {
    @service time;

    constructor() {
      this.currentTime = this.time.now();
    }
  }
  ```

4. **We could not support injections during object construction.** Instead, we could
formalize the `init` hook as a separate hook from the constructor, which
semantically means “initialized by the container”. This may be a reasonable hook
for the framework to have and could be very useful, even if it seems a bit
confusing (e.g. what’s the difference between `init` and `constructor`?)
