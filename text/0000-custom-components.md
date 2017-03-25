- Start Date: 2017-03-13
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

This RFC aims to expose a _low-level primitive_ for defining _custom
components_. This API will allow addon authors to implement special-purpose
component-based APIs (such as LiquidFire's animation helpers or low-overhead
components for performance hotspots).

In the medium term, this API and expected future enhancements will enable the
Ember community to experiment with alternative component APIs outside of the
core framework, for example enabling the community to prototype "angle bracket
components" using public APIs outside of the core framework.

# Motivation

The ability to author reusable, composable components is a core features of
the Ember.js framework. Despite being a [last-minute addition](http://emberjs.com/blog/2013/06/23/ember-1-0-rc6.html)
to Ember.js 1.0, the `Ember.Component` API has proven itself to be an extremely
powerful programming model and has aged well over time into the primary unit of
composition in the Ember view layer.

That being said, the current component API (hereinafter "classic components")
does have some noticeable shortcomings. Over time, classic components have also
accumulated some cruft due to backwards compatibility constraints.

These problems led to the original "angle bracket components" proposal (see RFC
[#15](https://github.com/emberjs/rfcs/blob/master/text/0015-the-road-to-ember-2-0.md)
and [#60](https://github.com/emberjs/rfcs/pull/60)), which promised to address
these problems via the angle bracket invocation opt-in (i.e. `<foo-bar ...>`
instead of `{{foo-bar ...}}`).

Since the transition to the angle bracket invocation syntax was seen as a rare,
once-in-a-lifetime opportunity, it became very tempting for the Ember core team
and the wider community to debate all the problems, shortcomings and desirable
features in the classic components API and attempt to design solutions for all
of them.

While that discussion was very helpful in capturing constraints and guiding the
overall direction, designing that One True API™ in the abstract turned out to
be extremely difficult and ultimately undesirable. It also went against the
Ember philosophy that framework features should be extracted from applications
and designed iteratively with feedback from real-world usage.

Since those proposals, we have rewritten Ember's rendering engine from the ground
up (the "Glimmer 2" project). One of the goals of the Glimmer 2 effort was to
build first-class primitives for our view-layer features in the rendering engine.
As part of the process, we worked to rationalize these features and to re-think
the role of components in Ember.js. This exercise has brought plenty of new ideas
and constraints to the table.

The initial Glimmer 2 integration was completed in [November](http://emberjs.com/blog/2016/11/30/ember-2-10-released.html).
Since Ember.js 2.10, classic components have been re-implemented on top of the
Glimmer 2 primitives, and we are very happy with the results.

This approach yielded a number of very powerful and flexible primitives:
in addition to classic components, we were able to implement the `{{mount}}`,
`{{outlet}}` and `{{render}}` helpers from Ember as "components" under the hood.

Based on our experience, we believe it would be beneficial to open up these new
primitives to the wider community. Specifically, there are at least two clear
benefits that comes to mind:

First, this would unlock new capabilities for addon authors, allowing them to
build custom components tailored to specific scenarios that are underserved by
the general-purpose component APIs (e.g. Liquid Fire's animation helpers,
low-overhead components for performance hotspots). Having an escape valve
for these scenarios also allows us to focus primarily on the mainstream use cases
when designing the new component API ("angle bracket components").

Second, this API and expected future enhancements will enable the Ember community
to experiment with alternative component APIs, and allow us to prototype "angle
bracket components" outside of the core framework purely on top of exposed public
APIs.

Following the success of FastBoot and Engines, we believe the best way to design
angle bracket components is to first stablize the underlying primitives in the
core framework and then experiment with the surface API through an addon.

# Detailed design

## What is a component?

In today's programming model, a component is often viewed narrowly as a device
for handling and managing user-interactions ("UI widgets").

While components are indeed very useful for building widgets, it doesn't just
stop there. In front-end development, they serve a much broader role, allowing
you to break up large templates into smaller, well-encapsulated units.

For example, you might break up a blog post into a headline, byline, body,
and a list of comments. Each comment might be further broken down into
a headline, author card and contents.

From Glimmer's perspective, components are analogous to functions in
other programming languages. Some components are designed to be reusable
in many contexts, but it's also perfect normal to use them to break apart
large chunks of logic.

In the most general sense, a component takes inputs (positional arguments,
named arguments, blocks, etc.), can be invoked, may render some content,
and knows how to keep its content up to date as its inputs change. From
the outside, you don't need to know how these details are managed, you
just need to know its API (i.e. what inputs it expects).

When looking at components expansively, it's no surprise that things that
are not usually thought of as components are implemented as components
in the template layer (input and link helpers, engines, outlets). Similarly,
we expect that this API to be useful far beyond just "UI widgets".

## `ComponentDefinition`

This RFC introduces a new type of object in Ember.js, `ComponentDefinition`,
which defines a component that can be invoked from a template.

Like classic components, a `ComponentDefinition` must be registered with a
dasherized name (with at least one dash in the name). This allows them to be
easily distinguishable from regular property lookups and HTML elements. Once
registered, the component can be invoked by name like a regular classic
component (i.e. `{{foo-bar}}`, `{{foo-bar "positional" "and" named="args"}}`,
`{{#foo-bar with or without=args}}...{{/foo-bar}}` etc).

> **Open Question**: How should these objects be registered? (see the last
  section of this RFC)

> **Open Question**: How does this interact with the `{{component}}` helper
and the `(component)` feature? (see the last section of this RFC)

A `ComponentDefinition` object should satisfy the following interface:

```typescript
interface ComponentDefinition {
  name: string;
  layout: string;
  manager: string;
  capabilities: ComponentCapabilitiesMask;
  metadata?: any;
}
```

> **Open Question**: Should we require `ComponentDefinition` to inherit from a
provided super class (`Ember.ComponentDefinition.extend({ ... }`) or otherwise
be wrapped with a function call (`Ember.Component.definition({ ... })`)?

The *name* property should contain an identifier for this component, usually
the component's dasherized name. This is primarily used by debug tools (e.g.
Ember Inspector).

The *layout* property specifies which template should be rendered when the
component is invoked. For example, if "foo-bar" is specified, Ember will lookup
the template `template:components/foo-bar` from the resolver.

> **Open Question**: How does this interact with local lookup?

The *manager* property is a string key specifying the `ComponentManager` to use
with this component. For example, if "foo" is specified here, Ember will lookup
the component manager `component-manager:foo`. This will be described in more
detail in the sections below.

> **Note**: Specifying the manager as a lookup key allows the component manager
to receive injections.

The *capabilities* property specifies the optional features required by this
component. This will be described in more detail below.

Finally, there is an optional *metadata* property which component authors can
use to store arbitrary data. For example, it may include a *class* property to
specify which component class to use. The *metadata* property is ignored by
Ember but can be used by the `ComponentManager` to perform custom logic.

## `ComponentManager`

Whereas a `ComponentDefinition` describe the static property of the component,
a `ComponentManager` controls its runtime properties.

A basic `ComponentManager` satisfies the following interface:

```typescript
interface ComponentManager<T> {
  create(definition: ComponentDefinition, args: ComponentArguments): T;
  getContext(instance: T): any;
  update(instance: T, args: ComponentArguments): void;
}

interface ComponentArguments {
  positional: any[];
  named: Object;
}
```

> **Open Question**: Should we require `ComponentManager` to inherit from a
provided super class (`Ember.ComponentManager.extend({ ... }`)?

When Ember is about to render a component, Ember will lookup its component
manager (as described above) and call its *create* method with the component
defition and the *component arguments*.

The *component arguments* object is a snapshot of the arguments pased into the
component. It has a *positional* and *named* property which corresponds to an
array and object (dictionary) of the current argument values.

For example, for the following invocation:

```hbs
{{blog-post (titleize post.title) post.body author=post.author excerpt=true}}
```

Glimmer will look up the `blog-post` definition and call *create* on its
manager with the following `ComponentArguments`:

```javascript
{
  positional: [
    "Rails Is Omakase",
    "Lorem ipsum dolor sit amet, consectetur adipiscing elit..."
  ],
  named: {
    "author": #<User name="David Heinemeier Hansson", ...>,
    "excerpt": true
  }
}
```

The component manager *must not* mutate the *component arguments* object (and
the inner *positional* and *named* object/array) directly as they might be
pooled or reused by the system.

> **Note**: We should probably freeze them in debug mode.

Based on the information in the definition and the *component arguments*, the
manager should return a *component instance* from the *create* method. From
Ember's perspective, this could be any arbitrary value – it is only used for
the component manager's internal book-keeping. In practice, this value should
store enough information to represent the internal state of the component. For
these reasons, it is often referred to as the "opaque state bucket".

At a later time, before Ember is ready to render the template, the component
manager's *getContext* method will be called. It will receive the *component
instance* and return the context object for the template. The context binds
`{{this}}` inside the layout, which is also the root of implicit property
lookups (e.g. `{{foo}}`, `{{foo.bar}}` are equivalent to `{{this.foo}}` and
`{{this.foo.bar}}`).

In many cases, the component manager can simply return the *component instance*
from this hook:

```typescript
class MyManager implements ComponentManager<MyComponent> {
  create(definition: ComponentDefinition, args: ComponentArguments): MyComponent {
    ...
  }

  getContext(instance: MyComponent): MyComponent {
    return instance;
  }

  update(instance: MyComponent, args: ComponentArguments) {
    ...
  }
}
```

However, they could also choose derive a different value from the state bucket.
This pattern allows the component manager to hide internal metadata:

```typescript
interface MyStateBucket {
  instance: MyComponent;
  secret: any;
}

class MyManager implements ComponentManager<MyStateBucket> {
  create(definition: ComponentDefinition, args: ComponentArguments): MyStateBucket {
    ...
  }

  getContext({ instance }: MyStateBucket): MyComponent {
    return instance;
  }

  update({ instance, secret }: MyStateBucket, args: ComponentArguments) {
    ...
  }
}
```

Finally, when any of the *component arguments* have changed, Ember will call
the *update* method on the manager, passing the *component instance* and a
snapshot of the current *component arguments* value. This happens before the
template is revalidated, therefore any updates to the context object performed
here will be reflected in the template.

Ember does not provide a fine-grained change tracking system, so there is no
guarantee on how often *update* will be called. More precisely, if one of the
*component arguments* have changed to a different value, *update* _will_ be
called at least once; however, the reverse is not true – when *update* is
called, it does not guarantee that at least one of the *component arguments*
will have a different value.

For example, given the following invocation:

```hbs
{{my-component unit=(pluralize "cat" count=cats.count)}}
```

If `cats.count` changes from 3 to 4, the *pluralize* helper will return exactly
the same value (the string `"cats"`), therefore from the manager's perspective,
the *component arguments* have not changed. However, in today's implementation,
the *update* method will still be called in this case. Over time, we intend for
internal changes in the implementation to cause this method to be called less
often, and other changes may cause it to be called more often. As a result,
component managers should not rely on this detail.

Here is a simple example that puts all of these pieces together:

```typescript
interface MyStateBucket {
  immutable: boolean;
  instance: Object;
}

class MyManager implements ComponentManager<MyStateBucket> {
  // on initial invocation, turn the arguments into a per-instance state bucket
  // that contains the component instance and some other internal details
  create(definition: ComponentDefinition, args: ComponentArguments): MyStateBucket {
    let immutable = Math.random() > 0.5;
    let instance = getOwner(this).lookup(`component:${definition.metadata.class}`);

    instance.setProperties(args.named);

    return { immutable, instance };
  }

  // expose the component instance (but not the internal details) as
  // {{this}} in the template
  getContext({ instance }: MyStateBucket): any {
    return instance;
  }

  // when update is called, update the properties on the component with the
  // new values of the named arguments.
  update({ immutable, instance }: MyStateBucket, args: ComponentArguments) {
    if (!immutable) {
      instance.setProperties(args.named);
    }
  }
}
```

While each invocable component needs its own `ComponentDefinition`, a single
`ComponentManager` instance can create and manage many *component instances*
(and this is why all of the hooks takes the *component instance* as the first
parameter). In fact, all classic components in Ember today share the same
component manager. Therefore, it is important to follow the pattern of keeping
your component's state in the state bucket, rather than storing it on the
manager itself.

That being said, in some rare cases, it might make sense to store some shared
states on the manager. For example, a component manager might want to maintain
a pool of *component instances* and reuse them when possible (e.g. `{{sustain}}`
in flexi). In this case, it would make sense to store the components pool on
the manager:

```typescript
class PooledManager implements ComponentManager<Object> {

  private pools = {};

  create({ metadata }: ComponentDefinition, args: ComponentArguments): Object {
    let pool = this.pools[metadata.name];
    let instance;

    if (pool && pool.length) {
      instance = pool.pop();
    } else {
      instance = getOwner(this).lookup(`component:${metadata.class}`);
    }

    instance.setProperties(args.named);

    return instance;
  }

  //...

}
```

## `ComponentCapabilitiesMask`

The *capabilities* property on a `ComponentDefinition` describes the optional
features required by this component. There are several reasons for this design.

First of all, performance-wise, this allows us to make the costs associated
with individual features PAYGO (pay-as-you-go). The Glimmer architecture makes
it possible to break down the work of component invocation into small pieces.

For example, a "template-only component" doesn't need Glimmer to invoke any
of the hooks related to the lifecycle management of the *component instance*.
Therefore, in an ideal world, they should not pay the cost associated with
invoking those hooks.

To accomplish this, we designed the component manager to only expose a minimal
set of features by default and allow individual components to opt-in to additional
features as needed. The capabilities mask is the mechanism for components to
signal that.

Second, it allows us to introduce new features and make improvements to this
API without breaking existing code. In addition to enumerating the required
features, the `ComponentCapabilitiesMask` also encodes the version of Ember.js
the `ComponentDefinition`/`ComponentManager` was developed for. This makes it
possible to, say, make angle bracket invocation the default for new components
but preserve the current default ("curly" invocation) for existing components
that were targeting and older API revision.

Ember.js will provide a function to construct a `ComponentCapabilitiesMask`.
Since this is a required property, at minimum, a component definition would
specify the following:

```js
import Ember from 'ember';

const { capabilities, definition } = Ember.Component;

export default definition({
  name: "foo-bar",
  capabilities: capabilities("2.13"),
  //...
});
```

The first argument to the function is a string specifying the Ember.js version
(up to the minor version) this component is targeting. As mentioned above, this
allows us to gracefully handle API changes between versions. This is important
because the raw, low-level APIs in Glimmer are still highly unstable.

Changes to the Glimmer APIs at this layer can significantly improve the overall
performance of the component system by changing its internal implementation.
Even though the surface API is still under active development, we would like to
start exposing the stable parts of this system to Ember addon authors as soon
as possible.

The version identifier in the capabilities mask acts as a safety net. It allows
Ember to absorb the churn and avoid breaking existing code.

For example, the API proposed in this RFC supplies **reified** *component
arguments* to the component manager's *create* and *update* methods, i.e. a
"snapshot" of the current *component arguments* values, which imposes some
unnecessary cost. Eventually, we would like to address this problem by moving
to a [reference](https://github.com/tildeio/glimmer/blob/master/guides/04-references.md)-based
API.

Imagine that, by the time Ember 2.14 comes around, we were able to stablize the
more performant reference-based APIs. Instead of the reified *component
arguments* snapshot, new components targeting Ember 2.14 (i.e. `capabilities("2.14")`)
will receive the reference-based *component arguments* objects instead.
However, existing components that were written before Ember 2.14 (e.g.
`capabilities("2.13")`) will continue to receive the reified *component
arguments* for backwards compatability.

The second argument allows components to opt-in to additional features that are
not enabled by default:

```js
import Ember from 'ember';

const { capabilities, definition } = Ember.Component;

export default definition({
  name: "foo-bar",
  capabilities: capabilities("2.13", {
    someFeature: true,
    anotherFeature: true
  }),
  //...
});
```

Under the hood, this will likely be implemented as a bit mask, but component
authors should treat it as an opaque value intended for internal use by the
framework only. There will not be any public APIs to introspect its value (e.g.
`isEnabled()`), so component managers should not rely on this to alter its
runtime behavior.

All features are additive. The default set of functionality for a particular
version is always the most lightweight component available at the time.

### Optional Features

Below is list of optional features supported in the initial implementation.
This list is intentionally kept minimal to limit the scope of this RFC.

Additional features (e.g. angle bracket invocation, references-based APIs,
providing access to the component element, event handling/delegation) should be
proposed in their own RFCs (they can even be proposed _in parallel_ to this
RFC) which allows for a more focused discussions around those topics.

* `asyncLifecycleHooks`

  When this feature is enabled, the `ComponentManager` should implement this
  extended interface:

  ```typescript
  interface WithAsyncHooks<T> extends ComponentManager<T> {
    didCreate(instance: T): void;
    didUpdate(instance: T): void;
  }
  ```

  These methods will be called with the *component instance* ("state bucket")
  *after the rendering process is complete*. This would be the right time to
  invoke any user callbacks, such as *didInsertElement* and *didRender* in the
  classic components API.

  It is guaranteed that the async callbacks will the invoked if and only if the
  synchronous APIs were invoked during rendering. For example, if *update* is
  called on a *component instance* during rendering, it is guaranteed *didUpdate*
  will be called. Conversely, if *didUpdate* is called, it is guaranteed that
  that *component instance* was updated during rendering (per the semantics of
  the *update* method described above).

  > **Open Question**: Should we provide ordering guarantee?

* `destructor`

  When this feature is enabled, the `ComponentManager` should implement this
  extended interface:

  ```typescript
  interface WithDestructor<T> extends ComponentManager<T> {
    destroy(instance: T): void;
  }
  ```

  The *destroy* method will be called with the *component instance* ("state
  bucket") when the component is no longer needed. This is intended for
  performing object-model level cleanup. It might also be useful for niche
  scenarios such as recycling *component instances* in the "component pool"
  example mentioned above.

  Because this RFC does not provide ways access to the component DOM tree, the
  timing relative to DOM teardown is undefined (i.e. whether this is called
  before or after the component's DOM tree is removed from the document).
  Therefore, this hook is **not** suitable for invoking DOM cleanup hooks such
  as *willDestroyElement* in the classic components API. We expect a subsequent
  RFC addressing DOM-related functionalities to clearify this issues (and
  most-likely provide a specialized hook for that purpose).

  Further, the exact timing for the call is also undefined. For example, the
  calls from several render loops might be batched together and deferred into a
  browser idle callback.

  > **Open Question**: Should we provide ordering guarantee?

* `viewSupport`

  When this feature is enabled, the `ComponentManager` should implement this
  extended interface:

  ```typescript
  interface WithViewSupport<T> extends ComponentManager<T> {
    getView(instance: T): Object;
  }
  ```

  The *getView* method will be called with the *component instance* ("state
  bucket") after the component has been created but before any children
  components in its layout (or yielded blocks) are created. The manager should
  return a view object corresponding to the *component instance* from this hook.
  A lot of times, this will just be the *component instance* itself, but this
  indirection allows you to hide (or intentionally expose) additional metadata.

  Ember will install a GUID on the returned object using `Ember.guidFor` and
  add the view to the view registry with the GUID. Currently, `Ember.guidFor`
  adds an internal property on the object to store the GUID, therefore the view
  cannot be frozen or sealed. This restriction might change in the future as
  we move to a `WeakMap`-based internal implementation.

  The view will also become visible to other components as a parent/child view.
  If the parent view is an `Ember.Component`, it will see your component in its
  *childViews* array. Similarly, when rendering other `Ember.Component` inside
  your comoponent's layout (or yielded blocks), they will see your component as
  their *parentView*.

  > **Open Question**: Will this break existing code?

  Note that these only applies to components that requests the *viewSupport*
  capability. Components that do not have *viewSupport* enabled will essentially
  be invisible in the Ember view hierarchy – they will not be added to the view
  registry and the parent/child view tree will essentially "jump through" these
  virtual views. At present, those components will also be invisible to debug
  tools like the Ember Inspector, but subsequent RFCs may change that.

  Components that are conceptually "advanced flow-control syntax", such as
  LiquidFire's *liquid-if*, most likely want to remain "virtual". On the other
  hand, general-purpose component toolkits probably want to participate in the
  view hierarchy like `Ember.Component`s do.

# Examples

> **Note**: For now, these examples side-steps the registration/resolution
> problem (see the "Unresolved Questions" section) by specifying the outcome
> of a particular container lookup (using the `type:name` convention) without
> addressing *how* they get in there in the first place.

## Isolated Partials

This example implements a very simple component API that acts more or less like
a partial in Ember. However, unlike partials, it will only have access to named
arguments that are explicitly passed in, instead of inheriting the caller's
scope.

Given the following `ComponentDefinition`s:

```javascript
// component-definition:site-header
{
  name: "site-header",
  layout: "site-header",
  manager: "isolated-partials",
  capabilities: capabilities("2.13")
}
```

```javascript
// component-definition:site-footer
{
  name: "site-footer",
  layout: "site-footer",
  manager: "isolated-partials",
  capabilities: capabilities("2.13")
}
```

```javascript
// component-definition:contact-us
{
  name: "contact-us",
  layout: "contact-us",
  manager: "isolated-partials",
  capabilities: capabilities("2.13")
}
```

...and the following templates:

```handlebars
{{!-- app/templates/application.hbs --}}

{{site-header}}

{{outlet}}

{{site-footer copyrightYear="2017" company=model}}
```

```handlebars
{{!-- app/templates/components/site-header.hbs --}}

<header>
  <h1>Welcome</h1>
</header>
```

```handlebars
{{!-- app/templates/components/site-footer.hbs --}}

<footer>
  &copy; {{copyrightYear}} {{company.name}}.

  {{contact-us tel=company.tel address=company.address}}
</footer>
```

```handlebars
{{!-- app/templates/components/contact-us.hbs --}}

<div class"contact-us">
  <h4>Contact Us</h4>
  <a href="tel:{{tel}}">{{tel}}</a>
  <address>{{address}}</address>
</div>
```

...and the following component manager:

```javascript
// component-manager:isolated-partials

class IsolatedPartialsManager extends ComponentManager<Object> {
  create(_definition: ComponentDefinition, args: ComponentArguments) {
    return { ...args.named };
  }

  getContext(instance: Object): Object {
    return instance;
  }

  update(instance: Object, args: ComponentArguments): void {
    Object.assign(instance, args.named);
  }
}
```

...it would render something like this:

```html
<header>
  <h1>Welcome</h1>
</header>

...

<footer>
  &copy; 2017 ACME Inc.

  <div class"contact-us">
    <h4>Contact Us</h4>
    <a href="tel:1-800-ACME-INC">1-800-ACME-INC</a>
    <address>100 Absolutely No Way, Portland, OR 98765</address>
  </div>
</footer>
```

This is probably most basic `ComponentManager` one could write. It essentially
just returns named arguments as `{{this}}` context and renders the component's
template. Nevertheless, it is a good example to understand the lifecycle of this
API.

## Pooled Components

This next example is inspired by Flexi's sustain feature to maintain a pool of
component instances.

Given the following `ComponentDefinition`s:

```javascript
// component-definition:list-item
{
  name: "list-item",
  layout: "list-item",
  manager: "pooled",
  capabilities: capabilities("2.13", {
    destructor: true,
    viewSupport: true
  })
}
```

...and the following templates:

```handlebars
{{!-- app/templates/application.hbs --}}

{{#each key="id" as |item|}}
  {{list-item item=item}}
{{/each}}
```

```handlebars
{{!-- app/templates/components/list-item.hbs --}}

<li id="{{item.id}}">{{item.value}}</li>
```

...and the following component manager:

```javascript
// component-manager:pooled

interface PoolEntry {
  name: string;
  instance: Ember.Component;
  expiresAt: number;
}

class ComponentPool {
  private pools = dict<PoolEntry>();

  checkout(name: string): PoolEntry | null {
    let pool = this.pools[name];

    if (pool && pool.length) {
      return pool.pop();
    } else {
      return null;
    }
  }

  checkin(entry: PoolEntry) {
    let { name } = entry;
    let pool = this.pools[name];

    if (!pool) {
      pool = this.pools[name] = [];
    }

    entry.expiresAt = Date.now() + EXPIRATION;

    pool.push(entry);
  }

  collect() {
    let now = Date.now();

    this.pools.forEach(pool => {
      let i = 0;
      let max = pool.length;

      while (i < max) {
        let entry = pool[i];

        if (entry.expiresAt < now) {
          entry.instance.destroy();
          pool.splice(i, 1);
          max--;
        } else {
          i++;
        }
      }
    });
  }
}

class PooledManager extends ComponentManager<PoolEntry> {

  private pool = new ComponentPool();

  create({ metadata }: ComponentDefinition, args: ComponentArguments) {
    let name = metadata.class;
    let entry = pool.checkout(name);
    let instance, isRecycled;

    if (entry) {
      instance = entry.instance;
      isRecycled = true;
    } else {
      instance = getOwner(this).lookup(`component:${name}`);
      isRecycled = false;
      entry = { name, instace, expiresAt: NaN };
    }

    instance.setProperties(args.named);

    if (isRecycled) {
      instance.didUpdateAttrs();
    } else {
      instance.didInitAttrs();
    }

    instance.didReceiveAttrs();

    return entry;
  }

  getContext({ instance }: PoolEntry): Ember.Component {
    return instance;
  }

  getView({ instance }): PoolEntry): Ember.Component {
    return instance;
  }

  update({ instance }: PoolEntry, args: ComponentArguments): void {
    instance.setProperties(args.named);
    instance.didUpdateAttrs();
    instance.didReceiveAttrs();
  }

  destroy(entry: PoolEntry) {
    this.pool.checkin(entry);
  }
}
```

When a `{{list-entry}}` component is rendered, the manager first check if there
are any recycled instances it could reuse. If not, it creates a new instance of
that component as usual. Either way, it calls `Ember.setProperties` on the
component instance to assign or update its properties based on the supplied
*component arguments*.

When the component is no longer needed, instead of destroying the component, it
checks it into a pool with an expiration timestamp. If another instance of
`{{list-entry}}` is rendered before the expiration, the component instance will
be reused, otherwise, it will eventually be destroyed through a scheduled GC task.

This examples shows how a custom component can request additional capability
(`destructor` and `viewSupport` in this case). It also shows an example of the
"state bucket" pattern – the component manager returns `entry.instance` as this
`{{this}}` context, allowing it to hide the internal `expiresAt` timestamp. It
also implements a subset of the `Ember.Component` hooks for *illustrative purpose*,
although the semantics in this **low-fi** implementation does not exactly match
the proper Ember semantics (**DO NOT** copy this code into an addon).

# How We Teach This

What is proposed in this RFC is a *low-level* primitive. We do not expect most
users to interact with this layer directly. Instead, they are expected to go
through additional conveniences provided by addon authors.

For addon authors, we need to teach "components as an abstraction/encapsulation
tool" as described above. For end users, we need to make sure using third-party
component toolkits mostly "feels" like using the first-class component APIs.

At present, the classic components APIs is still the primary, recommended path
for almost all use cases. This is the API that we should teach new users, so we
do not expect the guides need to be updated for this feature (at least not the
components section).

For documentation purposes, each Ember.js release will only document the latest
component manager API. The documentation will also include the steps needed to
upgrade, as well as a list of new capabilities added since the last version.
Documentation for a specific version of the component manager API can be viewed
from the versioned documentation site.

# Drawbacks

In the long term, there is a possiblity that while we work on the new component
API, the community will create a full fleet of competing third-party component
toolkits, thereby fragmentating the Ember ecosystem. However, given the Ember
community's love for conventions, this seems unlikely. We expect this to play
out similar to the data-persistence story – there will be a primary way to do
things (Ember Data), but there are also plenty of other alternatives catering
to niche use cases that are underserved by Ember Data.

Also, because apps can mix and match component styles, it's possible for a
library like smoke-and-mirrors or Liquid Fire to take advantage of the
enhanced functionality internally without leaking those implementation
details to applications.

# Alternatives

Instead of focusing on exposing enough low-level primitives to build the new
components API, we could just focus on building out the user-facing APIs
without rationalizing or exposing the underlying primitives.

# Unresolved Questions

There are a few minor inline open questions throughout the RFC. However, at
least two of them are worth repeating here.

1. `ComponentDefinition` registration and resolution

   There are (at least) two distinct use cases for the APIs proposed in this
   RFC.

   The first use case is for addons like LiquidFire and Flexi to implement
   advanced "helpers" more efficiently and tap into low-level features that are
   not exposed in the classic components API.

   For this use case, since there are only a small and finite amount of these
   helpers in each addon and that addon authors are relatively advanced users,
   the registration of these components does not have to be very egonormic.

   For example, we could allow addon authors to supply component definitions in
   `/{app,addon}/component-definitions/foo-bar.js` and component managers in
   `/{app,addon}/component-managers/my-manager.js`.

   > **Note**: we might need to restrict the definition files to "universal
   JavaScript" and disallow importing arbitrary files, see the build-time
   compilation discussion below.

   The second use case is to allow addons to develop an alternative component
   API for their users to extend and build on – a component SDK if you will.
   This would allow, among other things, the Ember core team to develop and
   iterate on the new components API ("angle bracket components") as an addon.

   Since this use case is targeting regular Ember users, and creating new
   components is a relatively common part of the day-to-day development
   workflow, the egonormics issue is much more important. Addons that chose to
   go down this route should be able to provide a developer experience similar
   to using classic components today. For instance, it would be unacceptable
   if developers have to create an `app/component-definitions/foo-bar.js` file
   for every custom component they create in addition to the template/JS file.

   One solution to this problem is to run introspection on the component's JS
   file. For example, we can require the exported value (usually the component
   class) from `app/components/foo-bar.js` to contain a *definition* static
   property that points to the component definition for that component.

   This solution has two major issues.

   First, one of the goals for the modules unification effort is to enable
   build-time template compilation. Currently, Glimmer does a lot of runtime
   triage due to syntatic ambiguities such as `{{foo-bar}}`, which could
   resolve into either a helper invocation, a component invocation or a
   property lookup depending on what is registered in the application. (In the
   future, `<foo-bar>` would have a similar issue since it could either be a
   component invocation or just a regular custom element/web component.)

   One of the side goals/hopes with modules unification effort is to move more
   of these compilation to build time by supplying a "build-time resolver" to
   answer these questions for the Glimmer compiler. For that to work, the
   "build-time resolver" would have to rely mostly on static information, such
   as file system entries, because most Ember application files would not load
   successfully without a full browser environment. (In general, to evaluate an
   app file while *in the middle of* compiling the app poses many challenges –
   in a lot of cases it is simply impossible.)

   By introducing custom components, the Glimmer compiler would need access to
   the component definitions (specifically, the capabilities mask) in order to
   emit the right compilation for the component. Therefore, if the only way to
   access the component definition is to evaluate the component JS file, that
   would pretty much make build-time compilation impossible.

   More importantly, the second issue with this approach is that because we are
   planning to move the component's tag into the template (as opposed to having
   *tagName*, *attributeBindings* and friends on the component class – see the
   previous angle bracket components RFCs), we expect template-only components
   (i.e. components without a JS file) to be relatively common. Therefore, in a
   lot of cases, there won't even be a JS class to stash the definition on to,
   making this solution a non-starter.

   Naturally, the solution seems to be that we should attach this information
   to the templates rather than the JS files. There are two proposals for this
   so far. The first proposal is to put this information in the filenames. For
   example, `foo-bar.glimmer.hbs` or `foo-bar.lite.hbs`. The second proposal is
   to put this infomration inside the body of the template as a "pragma", such
   as `{{!use glimmer}}`, `{{!use lite}}` (insert syntax bikeshed here). In
   both cases, addons can register a factory for the component definitions
   based on a string key ("glimmer" and "lite" in the examples).

2. Interactions with component helpers

   It is unclear how custom components should interact with the component
   helper feature.

   While there are indeed a lot of open questions on how this feature should
   work with angle bracket invocations, since this RFC does not introduce angle
   bracket invocation syntax, those questions can be deferred until a future
   RFC comes along to introduce that capability.

   However, while the `{{component}}` helper is implemented natively in Glimmer
   and should have no problem handling custom components (since all components
   are "custom" from Glimmer's perspective), it is unclear how it should work
   with the "closure" `(component)` helper.

   Currently, the "closure component" feature is not implemented natively in
   Glimmer. Instead, Glimmer currently exposes some low-level hooks to Ember's
   classic components manager, which allows it to implement its "closure"
   semantics. These hooks are unstable and might go away in future refactors.

   In the long run, we would like to implement this feature natively in Glimmer
   so that it would "just work" with custom components. However, the current
   "closure" semantics (specifically the ways it handles arguments currying) in
   Ember is quite specific to the classic components API and are somewhat
   counter-intuitive in some cases (the jury is still out, these issues might
   simply be mistakes in the current implementation).

   It seems unlikely that these issues would be resolved in the timeframe of
   this RFC (at minimum, it seems unwise to couple the two). Therefore, there
   are a few options here – 1) we can disallow using component helpers with
   custom components altogether for the MVP and relax that restriction in the
   future; 2) we can allow only `{{component}}` helper but not `(component)`;
   3) we can allow both `{{component}}` and `(component)`, *however*, we will
   disallow passing extra arguments to the `(component)` helper.

# Follow-up RFCs

We expect a few follow-up RFCs to introduce additional capabilities that are
not included in this minimal proposal. These RFCs can run either in parallel
to this RFC or be submitted after this initial RFC has been implemented and
tested in the wild.

1. Expose a way to get access to the component's DOM structure, such as its
   element and bounds. This RFC would also need to introduce a hook for DOM
   teardown and address how event handling/delegation would work.

2. Expose a way to get access to the [reference][1]-based APIs. This could
   include the ability to customize the component's "tag" ([validator][2]).

   [1]: https://github.com/tildeio/glimmer/blob/master/guides/04-references.md
   [2]: https://github.com/tildeio/glimmer/blob/master/guides/05-validators.md

3. Expose additional features that are used to implement classic components,
   `{{outlet}}` and other built-in components, such as layout customizations,
   and dynamic scope access.

4. Allow angle bracket invocation.
