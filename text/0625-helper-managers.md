---
Start Date: 2020-04-28
Relevant Team(s): Ember.js
RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
Tracking: (leave this empty)

---

# Helper Managers

## Summary

Provides a low-level primitive for defining helpers.

## Motivation

Helpers are a valuable template construct in Ember. They have a number of
benefits that come from having their lifecycle being managed by the template and
container, including:

1. Behavior can be _self-contained_. Some APIs need to run at multiple points in
   time based on a template's lifecycle, such as for a plugin that needs to be setup
   on initialization and torn down upon destruction. Doing this in components
   via lifecycle hooks usually forces users to split their API across multiple
   touch points in a component, which requires a lot of boilerplate and can make
   it difficult to understand how the whole system works together. Helpers
   provide a way for these shared concerns to be contained in a single location.

2. It doesn't require _multiple inheritance_. Alternatives for sharing behaviors
   that touch multiple parts of the lifecycle such as mixins and strategies like
   them create complicated inheritance hierarchies that can be difficult to debug.
   Helpers do not insert themselves into the inheritance hierarchy of a class,
   they are children of the template instead, which is much easier to reason
   about in practice.

3. They are highly _composable_. Helpers, like components, can be used multiple
   times in a template and can be used within `{{if}}` and `{{each}}` blocks.
   Combined with their ability to be hold a self contained lifecycle, this makes
   them a powerful tool for composing declarative behavior.

4. They can be _destroyed_. They tie in naturally to the destruction APIs that
   we have recently added to Ember, and that allows their lifecycle to be
   managed in a self contained way.

Helpers are currently the only template construct in Ember that do not have a
low-level primitive that is public. With components and modifiers, users can define
component managers and modifier managers respectively to create their own high
level APIs, but for helpers the only option currently is to use the high level
`helper()` wrapper function, or the `Helper` base class.

These APIs are beginning to show their age in Ember Octane, and unlocking
experimentation via a helper manager would allow us to begin designing a new
generation of helpers. Some possible areas to explore here would include:

* Using a native base class for helpers, instead of `EmberObject`
* Adding lifecycle hooks, similar to modifiers, to class-based helpers
* Adding the ability to inject services to functional helpers
* Allowing normal functions to operate as helpers

In addition, it would allow us to begin adding new functionality to helpers via
manager capabilities. This RFC proposes one such capability, `hasScheduledEffect`.

### Effect Helpers

Usually, template helpers are supposed to return a value. However, if a helper
returns `undefined` and rendered in a template, it will produce no output. This
can be used to accomplish a _side-effect_:

```js
// app/helpers/title.js
export default helper(([title]) => {
  window.document.title = title;
});
```

```hbs
{{title "My Document Title"}}
```

Addons such as [ember-page-title](https://github.com/adopted-ember-addons/ember-page-title)
use this to make helpers that can be added to the template to specify
_app behavior_ declaratively. This is a much better way to approach certain
types of behavior and APIs, compared to the alternative of using mixins and
lifecycle hooks to manage them.

However, this pattern has some issues today, mostly stemming from the fact that
they execute _during_ render, which is not an ideal time for side-effecting.
They can also be abused to modify app state, which can lead to difficult to
follow code paths reminiscent of observers. These issues stem from a mismatch
between two different goals, the goal of calculating a result or value, and the
goal of triggering side-effects.

The `hasScheduledEffect` capability would schedule side-effecting helpers to execute
_after_ render, and would _disable_ Ember's state mutations while they were
running. This would ensure that side-effecting helpers run at the optimal time,
and do not enable antipatterns and complicated codepaths.

## Detailed design

This RFC proposes adding the `setHelperManager` and `capabilities` APIs,
imported from `@ember/helper`. Like `setComponentManager` and
`setModifierManager`, `setHelperManager` receives a callback that is passed
the owner, and returns an instance of the helper manager. When a helper
definition is resolved by Ember, it will look up the manager recursively on the
definition's prototype chain until it finds a helper manager. If it does not
find one, it will throw an error.

```ts
// object is used here to mean any valid WeakMap key
type HelperDefinition = object;

export declare function setHelperManager(
  factory: (owner: Owner) => HelperManager,
  definition: HelperDefinition
): HelperDefinition;
```

And like the `capabilities` functions for component and modifier managers, the
`capabilities` function for helper managers receives a version string as the
first parameter and a options object as the second with optional flags as
booleans. It produces an opaque `HelperCapabilities` object, which is assigned
to the helper manager.

```ts
interface HelperCapabilitiesOptions {
  hasValue?: boolean;
  hasDestroyable?: boolean;
  hasScheduledEffect?: boolean;
}

type HelperCapabilities = Opaque;

export declare function capabilities(
  // to be replaced with the version of Ember this lands in
  version: '3.21.0',
  options: HelperCapabilitiesOptions
): HelperCapabilities;
```

Helper managers themselves have the following interface:

```ts
interface HelperManager<HelperStateBucket> {
  capabilities: HelperCapabilities;

  createHelper(definition: HelperDefinition, args: TemplateArgs): HelperStateBucket;

  getValue?(bucket: HelperStateBucket): unknown;

  runEffect?(bucket: HelperStateBucket): void;

  getDestroyable?(bucket: HelperStateBucket): object;
}
```

Let's dig into these hooks one by one:

### Hooks

#### `createHelper`

`createHelper` is a required hook on the HelperManager interface. The hook is
passed the definition of the helper that is currently being created, and is
expected to return a _state bucket_. This state bucket is what represents the
current state of the helper, and will be passed to the other lifecycle hooks at
appropriate times. It is not necessarily related to the definition of the
helper itself - for instance, you could return an object _containing_ an
instance of the helper:

```js
class MyManager {
  createHelper(Definition, args) {
    return {
      instance: new Definition(args);
    };
  }
}
```

This allows the manager to store metadata that it doesn't want to expose to the
user.

This hook is _not_ autotracked - changes to tracked values used within this hook
will _not_ result in a call to any of the other lifecycle hooks. This is because
it is unclear what should happen if it invalidates, and rather than make a
decision at this point, the initial API is aiming to allow as much expressivity
as possible. This could change in the future with changes to capabilities and
their behaviors.

If users do want to autotrack some values used during construction, they can
either create the instance of the helper in `runEffect` or `getValue`, or they
can use the `cache` API to autotrack the `createHelper` hook themselves. This
provides maximum flexibility and expressiveness to manager authors.

This hook has the following timing semantics:

**Always**
- called as discovered during DOM construction
- called in definition order in the template

#### `getValue`

`getValue` is an optional hook that should return the value of the helper. This
is the value that is returned from the helper and passed into the template.

This hook is called when the value is requested from the helper (e.g. when the
template is rendering and the helper value is needed). The hook is autotracked,
and will rerun whenever any tracked values used inside of it are updated.
Otherwise it does not rerun.

> Note: This means that arguments which are not _consumed_ within the hook will
> not trigger updates.

This hook is only called for helpers with the `hasValue` capability enabled.
This hook has the following timing semantics:

**Always**
- called the first time the helper value is requested
- called after autotracked state has changed

**Never**
- called if the `hasValue` capability is disabled

#### `runEffect`

`runEffect` is an optional hook that should run the effect that the helper is
applying, setting it up or updating it.

This hook is scheduled to be called some time after render and prior to paint.
There is not a guaranteed, 1-to-1 relationship between a render pass and this
hook firing. For instance, multiple render passes could occur, and the hook may
only trigger once. It may also never trigger if it was dirtied in one render
pass and then destroyed in the next.

The hook is autotracked, and will rerun whenever any tracked values used inside
of it are updated. Otherwise it does not rerun.

The hook is also run during a time period where state mutations are _disabled_
in Ember. Any tracked state mutation will throw an error during this time,
including changes to tracked properties, changes made using `Ember.set`, updates
to computed properties, etc. This is meant to prevent infinite rerenders and
other antipatterns.

This hook is only called for helpers with the `hasScheduledEffect` capability
enabled. This hook is also not called in SSR currently, though this could be
added as a capability in the future. It has the following timing semantics:

**Always**
- called after the helper was first created, if the helper has not been
  destroyed since creation
- called after autotracked state has changed, if the helper has not been
  destroyed during render

**Never**
- called if the `hasScheduledEffect` capability is disabled
- called in SSR

#### `getDestroyable`

`getDestroyable` is an optional hook that users can use to register a
destroyable object for the helper. This destroyable will be registered to the
containing block or template parent, and will be destroyed when it is destroyed.
See the [Destroyables RFC](https://github.com/emberjs/rfcs/blob/master/text/0580-destroyables.md)
for more details.

`getDestroyable` is only called if the `hasDestroyable` capability is enabled.

This hook has the following timing semantics:

**Always**
- called immediately after the `createHelper` hook is called

**Never**
- called if the `hasDestroyable` capability is disabled

### Capabilities

There are three proposed capabilities for helper managers:

* `hasDestroyable`
* `hasValue`
* `hasScheduledEffect`

Out of these capabilities, one of `hasScheduledEffect` or `hasValue` _must_ be
enabled. The other must _not_ be enabled, meaning they are mutually exclusive.

#### `hasDestroyable`

- Default value: false

Determines if the helper has a destroyable to include in the destructor
hierarchy. If enabled, the `getDestroyable` hook will be called, and its result
will be associated with the destroyable parent block.

#### `hasValue`

- Default value: false

Determines if the helper has a value which can be used externally. The helper's
`getValue` hook will be run whenever the value of the helper is accessed if this
capability is enabled.

#### `hasScheduledEffect`

- Default value: false

Determines if the helper has a scheduled effect. If enabled, the helper's
`runEffect` hook will run after render, and will not allow any type of state
mutation when running.

### Scheduled Helpers Timing

Scheduled helpers run their effects after render, and after modifiers have been
applied for a given render, but before paint. The exact timing may shift around,
and may or may not correspond to a single rendering pass in cases where there
are multiple rendering passes in a single paint.

In the future different timings may be added as options for scheduling. For
instance, a timing to call the effect using `requestIdleCallback`, when the
browser has finished rendering and handling higher priority work, could be
added. However, this is out of scope for this RFC.

## How we teach this

Helper managers are a low-level construct that is generally only meant to be
used by experts and addon authors. As such, it will only be taught through API
documentation. In addition, for precision and clarity, the API docs will include
snippets of TypeScript interfaces where appropriate.

### API Docs

#### `setHelperManager`

Sets the helper manager for an object or function.

```js
setHelperManager((owner) => new ClassHelperManager(owner), Helper)
```

When a value is used as a helper in a template, the helper manager is looked up
on the object by walking up its prototype chain and finding the first helper
manager. This manager then receives the value and can create and manage an
instance of a helper from it. This provides a layer of indirection that allows
users to design high-level helper APIs, without Ember needing to worry about the
details. High-level APIs can be experimented with and iterated on while the
core of Ember helpers remains stable, and new APIs can be introduced gradually
over time to existing code bases.

`setHelperManager` receives two arguments:

1. A factory function, which receives the `owner` and returns an instance of a
   helper manager.
2. A helper definition, which is the object or function to associate the factory function with.

The first time the object is looked up, the factory function will be called to
create the helper manager. It will be cached, and in subsequent lookups the
cached helper manager will be used instead.

Only one helper manager is guaranteed to exist per `owner` and per usage of
`setHelperManager`, so many helpers will end up using the same instance of the
helper manager. As such, you should only store state that is related to the
manager itself. If you want to store state specific to a particular helper
definition, you should assign a unique helper manager to that helper. In
general, most managers should either be stateless, or only have the `owner` they
were created with as state.

Helper managers must fulfill the following interface (This example uses
[TypeScript interfaces](https://www.typescriptlang.org/docs/handbook/interfaces.html)
for precision, you do not need to write helper managers using TypeScript):

```ts
interface HelperManager<HelperStateBucket> {
  capabilities: HelperCapabilities;

  createHelper(definition: HelperDefinition, args: TemplateArgs): HelperStateBucket;

  getValue?(bucket: HelperStateBucket): unknown;

  runEffect?(bucket: HelperStateBucket): void;

  getDestroyable?(bucket: HelperStateBucket): object;
}
```

The capabilities property _must_ be provided using the `capabilities()` function
imported from the same module as `setHelperManager`:

```js
import { capabilities } from '@ember/helper';

class MyHelperManager {
  capabilities = capabilities('3.21.0', { hasValue: true });

  // ...snip...
}
```

Below is a description of each of the methods on the interface and their
functions.

> The remaining API docs should be copied from the descriptions in the Detailed
> Design section of this RFC.

#### `capabilities`

`capabilities` returns a capabilities configuration which can be used to modify
the behavior of the manager. Manager capabilities _must_ be provided using the
`capabilities` function, as the underlying implementation can change over time.

The first argument to capabilities is a version string, which is the version of
Ember that the capabilities were defined in. Ember can add new versions at any
time, and these may have entirely different behaviors, but it will not remove
old versions until the next major version.

```js
capabilities('3.x');
```

The second argument is an object of capabilities and boolean values indicating
whether they are enabled or disabled.

```js
capabilities('3.x', {
  hasValue: true,
  hasDestructor: true,
});
```

If no value is specified, then the default value will be used.

##### `3.x` capabilities

> The remaining API docs should be copied from the descriptions in the Detailed
> Design section of this RFC. 3.x above should be replaced with the version that
> helper managers are initially released in.

## Drawbacks

- Adds a layer of indirection to helpers, which could add to complexity and
  cost in terms of performance. This isn't likely, as we haven't seen this
  happen with other managers we've introduced.

## Alternatives

- We could continue using the current helper APIs, and try to incrementally
  migrate them to only use native classes. This wouldn't match the strategy
  we've taken with other template constructs, like components and modifiers, and
  would result in less ability for the community to experiment and less
  flexibility if we chose to change helpers again in the future.

- The `hasScheduledEffect` capability could be broken out into a separate RFC.
  It is mostly separable, except for the impact it has on `getValue`. Value-less
  and effect-less helpers don't really make sense, so in isolation `getValue`
  would probably not be an optional hook, and the `hasValue` capability wouldn't
  exist.

  Capabilities can change from version to version, so this is still not a major
  issue, but it seems like it would be easier to add from the get go.

## Appendix

### Implementation of Current Helper APIs

The following is an implementation of the current helper APIs using this manager
API. There are two separate managers, one for class based helpers and one for
functional helpers:

```js
import EmberObject from '@ember/object';
import { tracked } from '@glimmer/tracking';
import { setHelperManager, capabilities } from '@ember/helper';

const RECOMPUTE = symbol();

export class Helper extends EmberObject {
  @tracked [RECOMPUTE];

  constructor(...args) {
    super(...args);

    registerDestructor(this, () => this.destroy());
  }

  recompute() {
    // update the value to force a recompute
    this[RECOMPUTED] = undefined;
  }
}

class ClassHelperManager {
  capabilities = capabilities({
    hasValue: true,
    hasDestroyable: true,
  });

  ownerInjection = {};

  constructor(owner) {
    setOwner(this.ownerInjection, owner);
  }

  createHelper(Definition, args) {
    let helper = Definition.create(this.ownerInjection);

    return { helper, args };
  }

  getValue({ helper, args }) {
    // Consume the RECOMPUTE tag, so if anyone ever
    // calls recompute() it'll force a recompute
    helper[RECOMPUTE];

    return helper.compute(args.positional, args.named);
  }

  getDestroyable({ helper }) {
    return helper;
  }
}

setHelperManager((owner) => new ClassHelperManager(owner), Helper.prototype);
```

```js
import { tracked } from '@glimmer/tracking';
import { setHelperManager, capabilities } from '@ember/helper';

class FunctionalHelperManager {
  capabilities = capabilities({
    hasValue: true,
  });

  createHelper(fn, args) {
    return { fn, args };
  }

  getValue({ fn, args }) {
    return fn(args.positional, args.named);
  }
}

const FUNCTIONAL_HELPER_MANAGER = () => new FunctionalHelperManager();

export function helper(fn) {
  setHelperManager(FUNCTIONAL_HELPER_MANAGER, fn);

  return fn;
}
```

### Implementation of Ember Page Title using Effects

This adapts the [ember-page-title](https://adopted-ember-addons.github.io/ember-page-title/)
addon to use the implementation proposed in this RFC. The biggest change to the
public API is that using `push` and `remove` directly schedules updates to the
title, so they can in theory be made public. The scheduling could also be moved
back to the helper itself to avoid that issue, this just cleans it up.

```js
// ember-page-title/addon/services/page-title-list

import Service, { inject as service } from '@ember/service';
import { getOwner } from '@ember/application';
import { scheduleOnce } from '@ember/run';

export default class PageTitleListService extends Service {
  @service headData;

  tokens = [];

  /**
    The default separator to use between tokens.
    @property defaultSeparator
    @default ' | '
   */
  defaultSeparator = ' | ';

  /**
    The default prepend value to use.
    @property defaultPrepend
    @default true
   */
  defaultPrepend = true;

  /**
    The default replace value to use.
    @property defaultReplace
    @default null
   */
  defaultReplace = null;

  constructor(owner) {
    super(owner);
    this._removeExistingTitleTag();

    let config = getOwner(this).resolveRegistration('config:environment');
    if (config.pageTitle) {
      ['separator', 'prepend', 'replace'].forEach((key) => {
        if (isPresent(config.pageTitle[key])) {
          set(this, `default${capitalize(key)}`, config.pageTitle[key]);
        }
      });
    }
  }

  applyTokenDefaults(token) {
    let {
      defaultSeparator,
      defaultPrepend,
      defaultReplace,
    } = this

    if (token.separator == null) {
      token.separator = defaultSeparator;
    }

    if (token.prepend == null && defaultPrepend != null) {
      token.prepend = defaultPrepend;
    }

    if (token.replace == null && defaultReplace != null) {
      token.replace = defaultReplace;
    }
  }

  inheritFromPrevious(token) {
    let { previous } = token;

    if (previous) {
      if (token.separator == null) {
        token.separator = previous.separator;
      }

      if (token.prepend == null) {
        token.prepend = previous.prepend;
      }
    }
  }

  push(token) {
    let { tokens } = this;
    let tokenForIdIndex = tokens.findIndex(({ id }), token.id === id);

    if (tokenForIdIndex) {
      let tokenForId = tokens[tokenForIdIndex];
      let { previous, next } = tokenForId;

      token.previous = previous;
      token.next = next;

      this.inheritFromPrevious(token);
      this.applyTokenDefaults(token);

      tokens.splice(tokenForIdIndex, 1, token);
      return;
    }

    let previous = tokens.slice(-1)[0];
    if (previous) {
      token.previous = previous;
      previous.next = token;
      this.inheritFromPrevious(token);
    }

    this.applyTokenDefaults(token);

    tokens.push(token);

    scheduleOnce('actions', this, this.updateTitle);
  }

  remove(id) {
    let { tokens } = this;

    let tokenIndex = tokens.findIndex(({ id }), token.id === id);
    let token = tokens[tokenIndex];
    let { next, previous } = token;

    if (next) {
      next.previous = previous;
    }

    if (previous) {
      previous.next = next;
    }

    token.previous = token.next = null;
    tokens.splice(tokenIndex, 1);

    scheduleOnce('actions', this, this.updateTitle);
  }

  updateTitle() {
    if (this.isDestroying || this.isDestroyed) return;

    this.headData.set('title', this.toString());
  }

  get visibleTokens() {
    let { tokens } = this;

    let replaceIndex = tokens.length;

    while (replaceIndex--) {
      if (tokens[replaceIndex].replace) {
        break;
      }
    }

    return tokens.slice(replaceIndex - 1);
  }

  get sortedTokens() {
    let visible = this.visibleTokens;
    let appending = true;
    let group = [];
    let groups = [group];
    let frontGroups = [];
    visible.forEach((token) => {
      if (token.front) {
        frontGroups.unshift(token);
      } else if (token.prepend) {
        if (appending) {
          appending = false;
          group = [];
          groups.push(group);
        }
        let lastToken = group[0];
        if (lastToken) {
          token = copy(token);
          token.separator = lastToken.separator;
        }
        group.unshift(token);
      } else {
        if (!appending) {
          appending = true;
          group = [];
          groups.push(group);
        }
        group.push(token);
      }
    });

    return frontGroups.concat(
      groups.reduce((E, group) => E.concat(group), [])
    );


  }

  toString() {
    let tokens = this.sortedTokens;

    return tokens
      .filter(token => Boolean(token.title))
      .map((token, index) => {
        if (index + 1 < tokens.length) {
          return token.title + token.separator;
        }

        return token.title;
      })
      .join('');
  }

  /**
   * Remove any existing title tags from the head.
   * @private
   */
  _removeExistingTitleTag() {
    if (this._hasFastboot()) {
      return;
    }

    let titles = document.getElementsByTagName('title');
    for (let i = 0; i < titles.length; i++) {
      let title = titles[i];
      title.parentNode.removeChild(title);
    }
  }

  _hasFastboot() {
    return !!getOwner(this).lookup('service:fastboot');
  }
}
```

```js
// ember-page-title/addons/helpers/title

import { setOwner } from '@ember/application';
import { inject as service } from '@ember/service';
import { setHelperManager, capabilities } from '@ember/helper';

class TitleHelperManager {
  capabilities = capabilities('3.21', {
    hasScheduledEffect: true,
  });

  constructor(owner) {
    this.owner = owner;
  }

  createHelper(Title, args) {
    return new Title(this.owner, args);
  }

  runEffect(instance) {
    instance.update();
  }

  getDestroyable({ instance }) {
    registerDestructor(instance, () => instance.teardown());

    return instance;
  }
}


export default class Title {
  @service pageTitleList;

  constructor(owner, args) {
    setOwner(this, owner);
    this.args = args;
    this.pageTitleList.push({ id: guidFor(this) });
  }

  update() {
    let token = assign({}, this.args.named, {
      id: guidFor(this),
      title: this.args.positional.join(''),
    });

    this.pageTitleList.push(token);
  },

  teardown() {
    this.pageTitleList.remove(guidFor(this));
  }
}

setHelperManager((owner) => new EffectHelperManager(owner), Title);
```

```hbs
{{title "Blog"}}
```
