- Start Date: 2014-10-24
- RFC PR:
- Ember Issue:

# Summary

Engines allow multiple logical applications to be composed together into
a single application from the user's perspective. They also allow addons
to provide encapsulated functionality normally associated with apps,
such as routes and templates that integrate seamlessly with the primary
app.

# Motivation

Large companies are increasingly adopting Ember.js to power their entire
product lines. Often, this means separate teams (sometimes distributed
around the world) working on the same app. Typically, responsibility is
shared by dividing the application into one or more "sections". How this
division is actually implemented varies from team to team.

Sometimes, each "section" will be a completely separate Ember app, with
a shared navigation bar allowing users to switch between each app. This
allows teams to work quickly without stepping on each others' toes, but
switching apps feels slow (especially compared to the normally speedy
route transitions in Ember) because the entire page must be thrown out,
then an entirely new set of the same assets downloaded and parsed.
Additionally, code sharing is largely accomplished via copy-and-paste.

Other times, the separation is enforced socially, with each team
claiming a section of the same app in the same repository.
Unfortunately, this approach leads to frequent conflicts around shared
resources, and feedback from tests gets slower and slower as test suites
grow in size.

Engines allow each section of an application to be developed in
isolation, including separate test suites and naming conventions. These
isolated units can be composed together into a single app from the
perspective of the user.

Additionally, engines allow addons to augment applications with
off-the-shelf fully-functional UI. For example, imagine an
authentication library that offers a prefab login screen that
application developers can opt in to, dramatically increasing the speed
at which prototypes can be developed.

Or imagine a product like Discourse allowing you to mount their Ember
app at `/forums/`, so the user needn't leave your app to read your
discussion forum.

# Detailed Design

Your app can gain access to an engine in one of two ways:

1. By placing an Ember CLI app inside another Ember CLI app's `engines/`
   directory (a sibling of `app/`).
2. By installing an Ember CLI addon that contains an `app` directory.

Engines are not loaded until the primary application mounts them
explicitly. The primary application uses the new `mount()` router DSL
method to mount an engine:

```js
Router.map(function() {
  this.mount('an-engine');
});
```

Engines' routes are prefixed with the name of the engine. For example,
let's write an engine that adds authentication, and includes a default
login and logout page. Imaginatively, we'll call this engine
`authentication`.

```js
// engines/authentication/router.js

import Ember from 'ember';

var Router = Ember.Router.extend();

Router.map(function() {
  this.resource('login');
  this.resource('logout');
});

export default Router;
```

Now, in the primary app, we mount the `authentication` engine:

```js
// app/router.js

import Ember from 'ember';

var Router = Ember.Router.extend();

Router.map(function() {
  this.mount('authentication');
});

export default Router;
```

Now the primary app gains two new URLs:

1. `/authentication/login`
2. `/authentication/logout`

If I want to customize the prefix used, I can pass options to `mount()`
that specify a path:

```js
this.mount('authentication', { path: 'auth' });
```

Now the URLs become:

1. `/auth/login`
2. `/auth/logout`

Engines otherwise behave like addons. Thanks to [the PR adding
namespaces to the Ember CLI resolver][namespaces], container keys can
specify an engine. For example, if the `authentication` engine contains
`engines/authentication/models/user.js`, this could be resolved via the
container with:

```js
container.lookup('authentication@model:user');
```

[namespaces]: https://github.com/stefanpenner/ember-resolver/pull/70

Other APIs in Ember would need to be extended to support namespaces to
take full advantage of this feature. For example, components that ship
with an engine could be accessed from the primary application like this:

```handlebars
{{authentication@login-form obscure-password=true}}
```

However, these changes are outside the scope of this RFC.

## Generator

Create a new engine by running `ember g engine-name`.

# Drawbacks

This RFC introduces the new concept of engines, which increases the
learning curve of the framework. However, I believe this issue is
mitigated by the fact that engines are an opt-in packaging around
existing concepts.

In the end, I believe that "engines" are just a small API for composing
existing concepts. And they can be introduced at the top of the
conceptual ladder, once users are comfortable with the basics of Ember,
and only for those working on large teams or distributing addons.

# Alternatives

One alternative is to simply allow addons to merge app-like assets
(templates, routes, etc.) with the primary app (e.g. via Broccoli's
`mergeTrees`). However, this is highly likely to lead to naming
collisions. By providing an isolated app that users can mount at a path
of their choosing, we can ensure that collisions are next to impossible.

# Unresolved questions

## Non-CLI Users

This RFC assumes Ember CLI. I would prefer to prove this out in Ember
CLI before locking down the public APIs/hooks the router exposes for
locating and mounting engines. Once this is done, however, we should
expose and document those hooks so users who cannot use Ember CLI for
whatever reason can still take advantage of composability.

## Dependencies (Shared vs. Isolated)

Because each engine is an isolated app, it will have its own set of
dependencies. Frequently, those dependencies will be the same as the
dependencies of the primary app. For example, it would be bad to ship
multiple copies of Ember, one for each engine. Or, imagine both the app
and an engine required a Bootstrap add-on that included CSS. It would
not do to include multiple (perhaps subtly different) copies of the same
CSS.

Ideally, dependencies would be deduplicated using a strategy described
by @wycats in [this Google doc](https://docs.google.com/a/tomdale.net/document/d/12CsR-zli5oP2TDWOef_-D28zjmbVD83hU4q9_VTk-9s/edit).

## App/Container/File Layout

Does each engine have its own container? Or does it register factories
with the host app's container?

Do engines subclass `Ember.Application`, or do we introduce something
like `Ember.Engine`?

How do engines use initializers? Can they inject into their own
namespace, or into the primary app also? How do you disambiguate?

Where does configuration go? Do engines put all of their app code in
`app/` like full-blown apps?
