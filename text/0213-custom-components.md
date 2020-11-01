---
Start Date: 2017-03-13
RFC PR: https://github.com/emberjs/rfcs/pull/213
Ember Issue: https://github.com/emberjs/ember.js/issues/16301

---

# Summary

This RFC aims to expose a _low-level primitive_ for defining _custom
components_.

This API will allow addon authors to provide special-purpose component base
classes that their users can subclass from in apps. These components are
invokable in templates just like any other Ember components (descendants of
`Ember.Component`) today.

# Motivation

The ability to author reusable, composable components is a core feature of
Ember.js. Despite being a [last-minute addition](http://emberjs.com/blog/2013/06/23/ember-1-0-rc6.html)
to Ember 1.0, the `Ember.Component` API and programming model has proven itself
to be an extremely versatile tool and has aged well over time into the primary
unit of composition in Ember's view layer.

That being said, the current component API (hereinafter "classic components")
does have some noticeable shortcomings. Over time, classic components have also
accumulated some cruft due to backwards compatibility constraints.

These problems led to the original "angle bracket components" proposal (see RFC
[#15](https://github.com/emberjs/rfcs/blob/master/text/0015-the-road-to-ember-2-0.md)
and [#60](https://github.com/emberjs/rfcs/pull/60)), which promised to address
these problems via the angle bracket invocation opt-in (i.e. `<foo-bar ...>`
instead of `{{foo-bar ...}}`).

Since the transition to the angle bracket invocation syntax was seen as a rare,
once-in-a-lifetime opportunity, it became very tempting to debate every single
shortcomings and missing features in the classic components API in the process
and attempt to design solutions for all of them.

While that discussion was very helpful in capturing constraints and guiding the
overall direction, designing that One True API™ in the abstract turned out to
be extremely difficult. It also went against our philosophy that framework
features should be extracted from applications and designed iteratively with
feedback from real-world usage.

Since that original proposal, we have rewritten Ember's rendering engine from
the ground up (the "Glimmer 2" project). One of the goals of the Glimmer 2
effort was to build first-class support for Ember's view-layer features into
the rendering engine. As part of the process, we worked to rationalize these
features and to re-think the role of components in Ember.js. This exercise has
brought plenty of new ideas and constraints to the table.

The initial Glimmer 2 integration was completed in [Ember 2.10](http://emberjs.com/blog/2016/11/30/ember-2-10-released.html).
As of that version, classic components have been re-implemented using the new
primitives provided by the rendering engine, and we are very happy with the
results.

This approach yielded a number of very powerful and flexible primitives:
in addition to classic components, we were able to implement Ember's
`{{mount}}`, `{{outlet}}` and `{{render}}` helpers as "components" under the
hood.

Based on our experience, we believe it would be beneficial to open up these new
primitives to the wider community. Specifically, there are at least two clear
benefits that comes to mind:

First, it provides addon authors fine-grained control over the exact behavior
and semantics of their components in cases where the general-purpose components
are a poor fit. For example, a low-overhead component designed to be used in
performance hotspot can opt-out of certain convinence features using this API.

Second, it allows the community to experiment with and iterate on alternative
component APIs outside of the core framework. Following the success of FastBoot
and Engines, we believe the best way to design the new "Glimmer Components" API
is to first stablize the underlying primitives in the core framework and
experiment with the surface API through an addon.

# Detailed design

This RFC introduces the concept of *component managers*. A component manager is
an object that is responsible for coordinating the lifecycle events that occurs
when invoking, rendering and re-rendering a component.

## Registering component managers

Component managers are registered with the `component-manger` type in the
application's registry. Similar to services, component managers are singleton
objects (i.e. `{ singleton: true, instantiate: true }`), meaning that Ember
will create and maintain (at most) one instance of each unique component
manager for every application instance.

To register a component manager, an addon will put it inside its `app` tree:

```js
// ember-basic-component/app/component-managers/basic.js

import EmberObject from '@ember/object';

export default EmberObject.extend({
  // ...
});
```

(Typically, the convention is for addons to define classes like this in its
`addon` tree and then re-export them from the `app` tree. For brevity, we will
just inline them in the `app` tree directly for the examples in this RFC.)

This allows the component manager to participate in the DI system – receiving
injections, using services, etc. Alternatively, component managers can also
be registered with imperative API. This could be useful for testing or opt-ing
out of the DI system. For example:

```js
// ember-basic-component/app/initializers/register-basic-component-manager.js

const MANAGER = {
  // ...
};

export function initialize(application) {
  // We want to use a POJO here, so we are opt-ing out of instantiation
  application.register('component-manager:basic', MANAGER, { instantiate: false });
}

export default {
  name: 'register-basic-component-manager',
  initialize
};
```

## Determining which component manager to use

For the purpose of this section, we will assume components with a JavaScript
file (such as `app/components/foo-bar.js` or the equivilant in "pods" and
[Module Unification](https://github.com/emberjs/rfcs/blob/master/text/0143-module-unification.md)
apps) and optionally a template file (`app/templates/components/foo-bar.hbs`
or equivilant). The example section has additional information about how this
relates to [template-only components](https://github.com/emberjs/rfcs/blob/master/text/0278-template-only-components.md).

When invoking the component `{{foo-bar ...}}`, Ember will first resolve the
component class (`component:foo-bar`, usually the `default` export from
`app/components/foo-bar.js`). Next, it will determine the appropiate component
manager to use based on the resolved component class.

Ember will provide a new API to assign the component manager for a component
class:

```js
// my-app/app/components/foo-bar.js

import EmberObject from '@ember/object';
import { setComponentManager } from '@ember/component';

export default setComponentManager('awesome', EmberObject.extend({
  // ...
}));
```

This tells Ember to use the `awesome` manager (`component-manager:awesome`) for
the `foo-bar` component. `setComponentManager` function returns the class.

In the future, this function can also be invoked as a decorator:

```js
// my-app/app/components/foo-bar.js

import EmberObject from '@ember/object';
import { componentManager } from '@ember/component';

export default @componentManager('awesome') EmberObject.extend({
  // ...
});
```

In reality, an app developer would never have to write this in their apps,
since the component manager would already be assigned on a super-class provided
by the framework or an addon. The `setComponentManager` function is essentially
a low-level API designed for addon authors and not intended to be used by app
developers.

For example, the `Ember.Component` class would have the `classic` component
manager pre-assigned, therefore the following code will continue to work as
intended:

```js
// my-app/app/components/foo-bar.js

import Component from '@ember/component';

export default Component.extend({
  // ...
});
```

Similarly, an addon can provided the following super-class:

```js
// ember-basic-component/addon/index.js

import EmberObject from '@ember/object';
import { componentManager } from '@ember/component';

export default setComponentManager('basic', EmberObject.extend({
  // ...
}));
```

With this, app developers can simply inherit from this in their app:

```js
// my-app/app/components/foo-bar.js

import BasicComponent from 'ember-basic-component';

export default BasicComponent.extend({
  // ...
});
```

Here, the `foo-bar` component would automatically inherit the `basic` component
manager from its super-class.

It is not advisable to override the component manager assigned by the framework
or an addon. Attempting to reassign the component manager when one is already
assinged on a super-class will be an error. If no component manager is set, it
will also result in a runtime error when invoking the component.

## Component Lifecycle

Back to the `{{foo-bar ...}}` example.

Once Ember has determined the component manager to use, it will be used to
manage the component's lifecycle.

### `createComponent`

The first step is to create an instance of the component. Ember will invoke the
component manager's `createComponent` method:

```javascript
// ember-basic-component/app/component-managers/basic.js

import EmberObject from '@ember/object';

export default EmberObject.extend({
  createComponent(factory, args) {
    return factory.create(args.named);
  },

  // ...
});
```

The `createComponent` method on the component manager is responsible for taking
the component's factory and the arguments passed to the component (the `...` in
`{{foo-bar ...}}`) and return an instantiated component.

The first argument passed to `createComponent` is the result returned from the
[`factoryFor`](https://github.com/emberjs/rfcs/blob/master/text/0150-factory-for.md)
API. It contains a `class` property, which gives you the the raw class (the
`default` export from `app/components/foo-bar.js`) and a `create` function that
can be used to instantiate the class with any registered injections, merging
them with any additional properties that are passed.

The second argument is a snapshot of the arguments passed to the component in
the template invocation, given in the following format:

```js
{
  positional: [ ... ],
  named: { ... }
}
```

For example, given the following invocation:

```hbs
{{blog-post (titleize post.title) post.body author=post.author excerpt=true}}
```

You will get the following as the second argument:

```js
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

The arguments object should not be mutated (e.g. `args.positional.pop()` is no
good). In development mode, it might be sealed/frozen to help prevent these
kind of mistakes.

### `getContext`

Once the component instance has been created, the next step is for Ember to
determine the `this` context to use when rendering the component's template by
calling the component manager's `getContext` method:

```js
// ember-basic-component/app/component-managers/basic.js

import EmberObject from '@ember/object';

export default EmberObject.extend({
  createComponent(factory, args) {
    return factory.create(args.named);
  },

  getContext(component) {
    return component;
  },

  // ...
});
```

The `getContext` method gets passed the component instance returned from
`createComponent` and should return the object that `{{this}}` should refer to
in the component's template, as well as for any "fallback" property lookups
such as `{{foo}}` where `foo` is neither a local variable or a helper (which
resolves to `{{this.foo}}` where `this` is here is the object returned by
`getContext`).

Typically, this method can simpliy return the component instance, as shown in
the example above. The reason this exists as a separate method is to enable the
so-called "state bucket" pattern which allows addon authors to attach extra
book-keeping metadata to the component:

```js
// ember-basic-component/app/component-managers/basic.js

import EmberObject from '@ember/object';

export default EmberObject.extend({
  createComponent(factory, args) {
    let metadata = { ... };
    let instance = factory.create(args.named);
    return { metadata, instance, ... };
  },

  getContext(bucket) {
    return bucket.instance;
  },

  // ...
});
```

Since the "state bucket", not the "context", is passed back to other hooks on
the component manager, this allows the component manager to access the extra
metadata but otherwise hide them from the app developers.

We will see an example that uses this pattern in a later section.

At this point, Ember will have gathered all the information it needs to render
the component's template, which will be rendered with ["Outer HTML" semantics](https://github.com/emberjs/rfcs/blob/master/text/0278-template-only-components.md).

In other words, the content of the template will be rendered as-is, without a
wrapper element (e.g. `<div id="ember1234" class="ember-view">...</div>`),
except for subclasses of `Ember.Component`, which will retain the current
legacy behavior (the internal `classic` manager uses private capabilities to
achieve that).

This API does not currently provide any way to fine-tune the rendering behavior
(such as dynamically changing the component's template) besides `getContext`,
but future iterations may introduce extra capabilities.

### `updateComponent`

When it comes time to re-render a component's template (usually because an
argument has changed), Ember will call the manager's `updateComponent` method
to give the manager an opportunity to reflect those changes on the component
instance, before performing the re-render:

```js
// ember-basic-component/app/component-managers/basic.js

import EmberObject from '@ember/object';

export default EmberObject.extend({
  createComponent(factory, args) {
    return factory.create(args.named);
  },

  getContext(component) {
    return component;
  },

  updateComponent(component, args) {
    component.setProperties(args.named);
  },

  // ...
});
```

The first argument passed to this method is the component instance returned by
`createComponent`. As mentioned above, using the "state bucket" pattern will
allow this hook to access the extra metadata:

```js
// ember-basic-component/app/component-managers/basic.js

import EmberObject from '@ember/object';

export default EmberObject.extend({
  createComponent(factory, args) {
    let metadata = { ... };
    let instance = factory.create(args.named);
    return { metadata, instance, ... };
  },

  getContext(bucket) {
    return bucket.instance;
  },

  updateComponent(bucket, args) {
    let { metadata, instance } = bucket;
    // do things with metadata
    instance.setProperties(args.named);
  },

  // ...
});
```

The second argument is a snapshot of the updated arguments, passed with the
same format as in `createComponent`. Note that there is no guarentee that
anything in the arguments object has _actually_ changed when this method is
called. For example, given:

```hbs
{{blog-post title=(uppercase post.title) ...}}
```

Imagine if `post.title` changed from `fOo BaR` to `FoO bAr`. Since the value
is passed through the `uppercase` helper, the component will see `FOO BAR` in
both cases.

Generally speaking, Ember does not provide any guarentee on how it determines
whether components need to be re-rendered, and the semantics may vary between
releases – i.e. this method may be called more or less often as the internals
changes. The _only_ guarentee is that if something _has_ changed, this method
will definitely be called.

If it is important to your component's programming model to _only_ notify the
component when there are actual changes, the manager is responsible for doing
the extra book-keeping.

For example:

```js
// ember-basic-component/index.js

import EmberObject from '@ember/object';
import { setComponentManager } from '@ember/component';

function NOOP() {}

export default setComponentManager('basic', EmberObject.extend({
  // Users of BasicComponent can override this hook to be notified when an
  // argument will change
  argumentWillChange: NOOP,

  // Users of BasicComponent can override this hook to be notified when an
  // argument will change
  argumentDidChange: NOOP,

  // ...
}));
```

```js
// ember-basic-component/app/component-managers/basic.js

import EmberObject from '@ember/object';

export default EmberObject.extend({
  createComponent(factory, args) {
    return {
      args: args.named,
      instance: factory.create(args.named)
    };
  },

  getContext(bucket) {
    return bucket.instance;
  },

  updateComponent(bucket, args) {
    let instance = bucket.instance;
    let oldArgs = bucket.args;
    let newArgs = args.named;
    let changed = false;

    // Since the arguments are coming from the template invocation, you can
    // generally assume that they have exactly the same keys. However, future
    // additions such as "splat arguments" in the template layer might change
    // that assumption.
    for (let key in oldArgs) {
      let oldValue = oldArgs[key];
      let newValue = newArgs[key];

      if (oldValue !== newValue) {
        instance.argumentWillChange(key, oldValue, newValue);
        instance.set(key, newValue);
        instance.argumentDidChange(key, oldValue, newValue);
      }
    }

    bucket.args = newArgs;
  },

  // ...
});
```

This example also shows when the "state bucket" pattern could be useful.

The return value of the `updateComponent` is ignored.

After calling the `updateComponent` method, Ember will update the component's
template to reflect any changes.

## Capabilities

In addition to the methods specified above, component managers are required to
have a `capabilities` property.  This property must be set to the result of
calling the `capabilities` function provided by Ember.

### Versioning

The first, mandatory, argument to the `capabilities` function is the component
manager API, which is denoted in the `${major}.${minor}` format, matching the
minimum Ember version this manager is targeting. For example:

```js
// ember-basic-component/app/component-managers/basic.js

import { capabilities } from '@ember/component';
import EmberObject from '@ember/object';

export default EmberObject.extend({
  capabilities: capabilities('3.2'),

  createComponent(factory, args) {
    return factory.create(args.named);
  },

  getContext(component) {
    return component;
  },

  updateComponent(component, args) {
    component.setProperties(args.named);
  }
});
```

This allows Ember to introduce new capabilities and make improvements to this
API without breaking existing code.

Here is a hypothical scenario for such a change:

1. Ember 3.2 implemented and shipped the component manager API as described in
   this RFC.

2. The `ember-basic-component` addon released version 1.0 with the component
   manager shown above (notably, it declared `capabilities('3.2')`).

3. In Ember 3.5, we determined that constructing the arguments object passed to
   the hooks is a major performance bottleneck, and changes the API to pass a
   "proxy" object with getter methods instead (e.g. `args.getPositional(0)` and
   `args.getNamed('foo')`).

   However, since Ember sees that the `basic` component manager is written to
   target the `3.2` API version, it will retain the old behavior and passes the
   old (more expensive) "reified" arguments object instead, to avoid breakage.

4. The `ember-basic-component` addon author would like to take advantage of
   this performance optimization, so it updates its component manager code to
   work with the arguments proxy and changes its capabilities declaration to
   `capabilities('3.5')` in version 2.0.

This system allows us to rapidly improve the API and take advantage of
underlying rendering engine features as soon as they become available.

Note that addon authors are not _required_ to update to the newer API.
Concretely, component manager APIs have the following support policy:

* API versions will continue to be supported in the same major release of
  Ember. As shown in the example above, `ember-basic-component` 1.0 (which
  targets component manager API version 3.2), will continue to work on
  Ember 3.5. However, the reverse is not true – component manager API version
  3.5 will (somewhat obviously) not work in Ember 3.2.

* In addition, to ensure a smooth transition path for addon authors and app
  developers across major releases, each Ember version will support (at least)
  the previous LTS version as of the release was made. For example, if 3.16 is
  the last LTS release of the 3.x series, the component manager API version
  3.16 will be supported by Ember 4.0 through 4.4, at minimum.

Addon authors can also choose to target multiple versions of the component
manager API using [ember-compatibility-helpers](https://github.com/pzuraq/ember-compatibility-helpers/):

```js
// ember-basic-component/app/component-managers/basic.js

import { gte } from 'ember-compatibility-helpers';

let ComponentManager;

if (gte('3.5')) {
  ComponentManager = EmberObject.extend({
    capabilities: capabilities('3.5'),

    // ...
  });
} else {
  ComponentManager = EmberObject.extend({
    capabilities: capabilities('3.2'),

    // ...
  });
}

export default ComponentManager;
```

Since the conditionals are resolved at build time, the irrevelant code will be
stripped from production builds, avoiding any deprecation warnings.

### Optional Features

The second, optional, argument to the `capabilities` function is an object
enumerating the optional features requested by the component manager.

In the hypothical example above, while the "reified" arguments objects may be
a little slower, they are certainly easier to work with, and the performance
may not matter to but the most performance critical components. A component
manager written for Ember 3.5 (again, only hypothically) and above would be
able to explicitly opt back into the old behavior like so:

```js
// ember-basic-component/app/component-managers/basic.js

import { capabilities } from '@ember/component';
import EmberObject from '@ember/object';

export default EmberObject.extend({
  capabilities: capabilities('3.5', {
    reifyArguments: true
  }),

  // ...
});
```

In general, we will aim to have the defaults set to as bare-bone as possible,
and allow the component managers to opt into the features they need in a PAYGO
(pay-as-you-go) manner, which aligns with the Glimmer VM philosophy. As the
rendering engine evolves, more and more feature will become optional.

## Optional Capabilities

The following optionally capabilities will be available with the first version
of the component manager API. We expect future RFCs to propose additional
capabilities within the framework provided by this initial RFC.

### Async Lifecycle Callbacks

When the `asyncLifecycleCallbacks` capability is set to `true`, the component
manager is expected to implement two additional methods: `didCreateComponent`
and `didUpdateComponent`.

`didCreateComponent` will be called after the component has been rendered the
first time, after the whole top-down rendering process is completed. Similarly,
`didUpdateComponent` will be called after the component has been updated, after
the whole top-down rendering process is completed. This would be the right time
to invoke any user callbacks, such as `didInsertElement` and `didRender` in the
classic components API.

These methods will be called with the component instance (the "state bucket"
returned by `createComponent`) as the only argument. The return value is
ignored.

These callbacks are called if and only if their synchronous APIs were invoked
during rendering. For example, if `updateComponent` was called on during
rendering (and it completed without errors), `didUpdateComponent` will always
be called. Conversely, if `didUpdateComponent` is called, you can infer that
the `updateComponent` was called on the same component instance during
rendering.

This API provides no guarentee about ordering with respect to siblings or
parent-child relationships.

### Destructors

When the `destructor` capability is set to `true`, the component manager is
expected to implement an additional method: `destroyComponent`.

`destroyComponent` will be called when the component is no longer needed. This
is intended for performing object-model level cleanup.

Because this RFC does not provide ways to access or observe the component's DOM
tree, the timing relative to DOM teardown is undefined (i.e. whether this is
called before or after the component's DOM tree is removed from the document).

Therefore, this hook is not suitable for invoking user callbacks intended for
performing DOM cleanup, such as `willDestroyElement` in the classic components
API. We expect a subsequent RFC addressing DOM-related functionalities to
clarify this issues or provide another specialized method for that purpose.

Similar to the other async lifecycle callbacks, this API provides no guarentee
about ordering with respect to siblings or parent-child relationships. Further,
the exact timing of the calls are also undefined. For example, the calls from
several render loops might be batched together and deferred into a browser idle
callback.

# Examples

## Basic Component Manager

Here is the simpliest end-to-end component manager example that uses a plain
`Ember.Object` super-class (as opposed to `Ember.Component`) with "Outer HTML"
semantics:

```js
// ember-basic-component/app/component-managers/basic.js

import { capabilities } from '@ember/component';
import EmberObject from '@ember/object';

export default EmberObject.extend({
  capabilities: capabilities('3.2', {
    destructor: true
  }),

  createComponent(factory, args) {
    return factory.create(args.named);
  },

  getContext(component) {
    return component;
  },

  updateComponent(component, args) {
    component.setProperties(args.named);
  },

  destroyComponent(component) {
    component.destroy();
  }
});
```

```js
// ember-basic-component/addon/index.js

import EmberObject from '@ember/object';
import { setComponentManager } from '@ember/component';

export default setComponentManager('basic', EmberObject.extend());
```

### Usage

```js
// my-app/app/components/x-counter.js

import BasicCompoment from 'ember-basic-component';

export default BasicCompoment.extend({
  init() {
    this.count = 0;
  },

  down() {
    this.decrementProperty('count');
  },

  up() {
    this.incrementProperty('count');
  }
});
```

```hbs
{{!-- my-app/app/templates/components/x-counter.hbs --}}

<div>
  <button {{action this.down}}>🔽</button>
  {{this.count}}
  <button {{action this.up}}>🔼</button>
</div>
```

## Template-only Components

This example implements a kind of component similar to what was proposed in the
[template-only components](https://github.com/emberjs/rfcs/blob/master/text/0278-template-only-components.md)
RFC.

Since the custom components API proposed in this RFC requires a JavaScript
files, we cannot implement true "template-only" components. We will need to
create a component JS file to export a dummy value, for the sole purpose of
indicating the component manager we want to use.

In practice, there is no need for an addon to implement this API, since it is
essentially re-implementing what the "template-only-glimmer-components"
optional feature does. Nevertheless, this example is useful for illustrative
purposes.

```js
// ember-template-only-component/app/component-managers/template-only.js

import { capabilities } from '@ember/component';
import EmberObject from '@ember/object';

export default EmberObject.extend({
  capabilities: capabilities('3.2'),

  createComponent() {
    return null
  },

  getContext() {
    return null;
  },

  updateComponent() {
    return;
  }
});
```

```js
// ember-template-only-component/addon/index.js

import { setComponentManager } from '@ember/component';

// Our `createComponent` method does not actually do anything with the factory,
// so we don't even need to export a class here, `{}` would work just fine.
export default setComponentManager('template-only', {});
```

### Usage

```js
// my-app/app/components/hello-world.js

import TemplateOnlyComponent from 'ember-template-only-component';

export default TemplateOnlyComponent;
```

```hbs
Hello world! I have no backing class! {{this}} would be <code>null</code>.
```

## Recycling Components

This example implements an API which maintain a pool of recycled component
instances to avoid allocation costs.

This example also make use of the "state bucket" pattern.

```js
// ember-component-pool/app/component-managers/pooled.js

import { capabilities } from '@ember/component';
import EmberObject from '@ember/object';

// How many instances to keep (per type/factory)
const LIMIT = 10;

export default EmberObject.extend({
  capabilities: capabilities('3.2', {
    destructor: true
  }),

  init() {
    this.pool = new Map();
  },

  createComponent(factory, args) {
    let instances = this.pool.get(factory);
    let instance;

    if (instances && instances.length > 0) {
      instance = instances.pop();
      instance.setProperties(args.named);
    } else {
      instance = factroy.create(args.named);
    }

    // We need to remember which factory does the instance belong to so we can
    // check it back into the pool later.
    return { factory, instance };
  },

  getContext({ instance }) {
    return instance;
  },

  updateComponent({ instance }, args) {
    instance.setProperties(args.named);
  },

  destroyComponent({ factory, instance }) {
    let instances;

    if (this.pool.has(factory)) {
      instances = this.pool.get(factory);
    } else {
      this.pool.set(factory, instances = []);
    }

    if (instances.length >= LIMIT) {
      instance.destroy();
    } else {
      // User hook to reset any state
      instance.willRecycle();
      instances.push(instance);
    }
  },

  // This is the `Ember.Object` lifecycle method, called when the component
  // manager instance _itself_ is being destroyed, not to be confused with
  // `destroyComponent`
  willDestroy() {
    for (let instances of this.pool.values()) {
      instances.forEach(instance => instance.destroy());
    }

    this.pool.clear();
  }
});
```

```js
// ember-component-pool/addon/index.js

import EmberObject from '@ember/object';
import { setComponentManager } from '@ember/component';

function NOOP() {}

export default setComponentManager('pooled', EmberObject.extend({
  // Override this to implement reset any state on the instance
  willRecycle(): NOOP,

  // ...
}));
```

# How We Teach This

What is proposed in this RFC is a *low-level* primitive. We do not expect most
users to interact with this layer directly. Instead, most users will simply
benefit from this feature by subclassing these special components provided by
addons.

At present, the classic components APIs is still the primary, recommended path
for almost all use cases. This is the API that we should teach new users, so we
do not expect the guides need to be updated for this feature (at least not the
components section).

For documentation purposes, each Ember.js release will only document the latest
component manager API, along with the available optional capabilities for that
realease. The documentation will also include the steps needed to upgrade from
the previous version. Documentation for a specific version of the component
manager API can be viewed from the versioned documentation site.

# Drawbacks

In the long term, there is a risk of fragmentating the Ember ecosystem with
many competing component APIs. However, given the Ember community's strong
desire for conventions, this seems unlikely. We expect this to play out similar
to the data-persistence story – there will be a primary way to do things (Ember
Data), but there are also plenty of other alternatives catering to niche use
cases that are underserved by Ember Data.

Also, because apps can mix and match component styles, it's possible for a
library like smoke-and-mirrors or Liquid Fire to take advantage of the
enhanced functionality internally without leaking those implementation
details to applications.

# Alternatives

Instead of focusing on exposing enough low-level primitives to build the new
components API, we could just focus on building out the user-facing APIs
without rationalizing or exposing the underlying primitives.

# Appendix

## Follow-up RFCs

We expect to rapidly iterate and improve the component manager API through the
RFC process and in-the-field usage/implementation experience. Here are a few
examples of additional capabilities that we hope to see proposed after this
initial (and intentionally minimal) proposal is finalized:

1. Expose a way to access to the component's DOM structure, such as its bounds.
   This RFC would also need to introduce a hook for DOM teardown and address
   how event handling/delegation would work.

2. Expose a way to access to the [reference][1]-based APIs. This could include
   the ability to customize the component's "tag" ([validator][2]).

   [1]: https://github.com/glimmerjs/glimmer-vm/blob/master/guides/04-references.md
   [2]: https://github.com/glimmerjs/glimmer-vm/blob/master/guides/05-validators.md

3. Expose additional features that are used to implement classic components,
   `{{outlet}}` and other built-in components, such as layout customizations,
   and dynamic scope access.

4. Angle bracket invocation.

## Using ES6 Classes

Although this RFC uses `Ember.Object` in the examples, it is not a "hard"
dependency.

### Using ES6 Classes For Components

The main interaction between the Ember object model and the component class
is through the DI system. Specifically, the factory function returned by
`factoryFor` (`factoryFor('component:foo-bar').create(...)`), which is passed
to the `createComponent` method on the component manager, assumes a static
`create` method on the class that takes the "property bag" and returns the
created instance.

Therefore, as long as your ES6 super-class provides such a function, it will
work with the rest of the system:

```js
// ember-basic-component/addon/index.js

import { setComponentManager } from '@ember/component';

class BasicComponent {
  static create(props) {
    return new this(props);
  }

  constructor(props) {
    // Do things with props, such as:
    Object.assign(this, props);
  }

  // ...
}

export default setComponentManager('basic', BasicComponent);
```

```js
// ember-basic-component/app/component-managers/basic.js

import { capabilities } from '@ember/component';
import EmberObject from '@ember/object';

export default EmberObject.extend({
  capabilities: capabilities('3.2'),

  createComponent(factory, args) {
    // This Just Works™ since we have a static create method on the class
    return factory.create(args.named);
  },

  // ...
});
```

```js
// my-app/app/components/foo-bar.js

import BasicCompoment from 'ember-basic-component';

export default class extends BasicCompoment {
  // ...
};
```

Alternatively, if you prefer not to add a static create method to your
super-class, you can also instantiate them in the component manager without
going through the DI system:

```js
// ember-basic-component/app/component-managers/basic.js

import { capabilities } from '@ember/component';
import EmberObject from '@ember/object';

export default EmberObject.extend({
  capabilities: capabilities('3.2'),

  createComponent(factory, args) {
    // This does not use the factory function, thus no longer require a static
    // create method on the class
    return new factory.class(args.named);
  },

  // ...
});
```

However, doing do will prevent your components from receiving injections (as
well as setting the appropiate owner, etc). Therefore, when possible, it is
better to go through the DI system's factory function.

### Using ES6 Classes For Component Managers

It is also possible to use ES6 classes for the component managers themselves.
The main interaction here is that they are automatically instantiated by the DI
system on-demand, which again assumes a static `create` method:

```js
// ember-basic-component/app/component-managers/basic.js

import { capabilities } from '@ember/component';

export default class BasicComponentManager {
  static create(props) {
    return new this(props);
  }

  constructor(props) {
    // Do things with props, such as:
    Object.assign(this, props);
  }

  capabilities = capabilities('3.2');

  // ...
};
```

Alternatively, as shown above, you can also register the component manager
with `{ instantiate: false }`:

```js
// ember-basic-component/app/initializers/register-basic-component-manager.js

import BasicComponentManager from 'ember-basic-component';

export function initialize(application) {
  application.register('component-manager:basic', new BasicComponentManager(), { instantiate: false });
}

export default {
  name: 'register-basic-component-manager',
  initialize
};
```

Note that this behaves a bit differently as the component manager instance is
shared across all application instances and is never destroyed, which might
affect stateful component managers such as the one shown in the "Recycling
Components" example above.
