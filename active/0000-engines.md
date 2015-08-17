- Start Date: 2014-10-24
- RFC PR:
- Ember Issue:

# Summary

Engines allow multiple logical applications to be composed together into a
single application from the user's perspective.

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

A more modular approach is to break off elements of a single application into
separate [addons](http://www.ember-cli.com/user-guide/#addons). Addons are
essentially mixins for [ember-cli](http://www.ember-cli.com/) applications. In
other words, the elements of an addon are merged with those of the application
that includes them. While addons allow for distributed development, testing, and
packaging, they do not provide the logical run-time separation required for
developing completely independent "sections" of an application. Addons must
function within the namespace, registry, and router of the application in which
they are included.

Engines provide an alternative to these approaches that allows for distributed
development, testing, and packaging, _as well as_ logical run-time separation.
Because engines are derived from applications, they can be just as
full-featured. Each has its own namespace, registry, and routing map. Even
though engines are isolated from the applications that contain them, the
boundaries between them allow for controlled sharing of resources.

The potential scope of engines is large enough that this feature merits
development and delivery in multiple phases. A minimum viable version could be
released sooner, which could be augmented with more advanced features later.

An initial release of engines could provide the following benefits:

* Distributed development - Engines can be developed and tested in isolation
  within their own Ember CLI projects and included by applications or other
  engines. Engines can be packaged and released as addons themselves.

* Integrated routing - Each engine should define a routing map that applications
  or other engines can mount at any point in their own routing maps.

* Clean boundaries - An engine can cooperate with its parents through a few
  explicit interfaces. Beyond these interfaces, engines and applications are
  isolated.

Subsequent releases of engines could allow for the following:

* Lazy loading - An engine could allow its parent to boot with only its routing
  map loaded. The rest of the engine could be loaded only as required (i.e.
  when a route in an engine is visited). This would allow applications to boot
  faster and limit their memory consumption.

* Namespaced access to engine resources from applications - This could open up
  the potential for applications to use, and extend, an engine's resources much
  like resources in other addons, but without the possibility of namespace
  collisions.

## Detailed design

Engines are very similar to regular applications: they can be developed in
isolation in Ember CLI, include addons, and contain all the same elements,
including routes, components, initializers, etc. The primary differences are
that an engine does not boot itself and an engine does not control the router.

### Engine internals

New `Engine` and `EngineInstance` classes will be introduced.

Applications and engines will share ancestry. It remains TBD whether
applications will subclass engines, or whether a common ancestor will be
introduced.

Engines and applications will share the same pattern for registry / container
ownership and encapsulation. Both will also have initializers and instance
initializers.

Engine instances will have access to their parent instances. An engine's parent
could be either an application or engine.

Engines will define their routes in a new `Ember.Routes` class. This class will
encapsulate the functionality provided by `Router#map`, and will be used
internally by `Ember.Router` as well (with no public interface changes of
course).

### Developing engines

Engines can be developed in isolation as Ember CLI addon projects or as part of
a parent application.

#### Engines as addons

Engines can be created as separate addon projects with:

```
ember engine <engine-name>
```

This will create a special form of an ember addon. The file structure will match
that of a standard addon, but will have an `engine` directory instead of an
`addon` directory.

Engines can be unit tested and can also be integration tested within a dummy
app, just like standard addons.

#### In-repo engines

An engine can be created within an existing application's project using a
special `in-repo-engine` generator (similar to the `in-repo-addon` generator):

```
ember g in-repo-engine <engine-name>
```

In-repo engines can be unit tested in isolation or integration testing with the
main application (instead of a dummy application).

> Note: In-repo addons currently are created in the `/lib` directory (e.g.
`/lib/my-addon`). Unit tests and integration tests are currently co-mingled with
tests for the main application. It's recommended that in-repo engines provide
better test separation than is provided for regular addons, and perhaps the
whole in-repo addon directory structure should be re-examined at the same time
in-repo engines are introduced.

#### Engine directory structure

An engine's directory will contain a file structure identical to the `app`
directory in a standard ember-cli application, with the following exceptions:

* `engine.js` instead of `app.js` - defines the `Engine` class and
  loads its initializers.

* `routes.js` instead of `router.js` - defines an engine's routing map in a
  `Routes` class.

### Installing engines

Engines developed as addons can be installed in an application just like any
other addon:

```
ember install <engine-name>
```

During development, you can use `npm link` to make your engine available in
another parent engine or application.

### Mounting engines

The new `mount()` router DSL method is used to mount an engine at a particular
"mount-point" in a route map.

For example, the following route map mounts the `discourse` engine at the
`/forum` path:

```
Router.map(function() {
  this.mount('discourse', {path: '/forum'});
});
```

> Note: If unspecified, `path` will match the name of the engine.

Calls to `mount` can be nested within routes. An engine can be mounted at
multiple routes, and each will represent a new instance of the engine to be
created.

### Loading phases

Engines can exist in several phases:

* Booted - an engine that's been installed in a parent application will have
  its dependencies loaded and its (non-instance) initializers invoked when the
  parent application boots.

* Mounted - an engine is considered "mounted" when it has been included by a
  router at one or more mount-points. The router inspects the engine's route
  map and is able to form URLs without fully loading the engine. Mounting is
  done as part of a router's initialization process.

* Instantiated - an engine is instantiated when a route is visited at or beyond
  its mount-point. An `EngineInstance` is instantiated and an engine's instance
  initializers are invoked.

Special `before` and `after` hooks could be added to application instance
initializers that allow them to be ordered relative to engine instance
initializers.

### Engine boundaries

Besides its routing map, an engine does not share any other resources with its
parent by default. Engines maintain their own registries and containers, which
ensure that they stay isolated. However, some explicit sharing of resources
between engines and parents should be allowed.

#### Engine / parent dependencies

Dependencies between engines and parents can be defined imperatively or
declaratively.

Imperative dependencies can be defined in an engine's instance initializers.
When an engine is instantiated, the `parent` property on its `EngineInstance` is
set to its parent instance (either an `ApplicationInstance` or
`EngineInstance`). Since the engine instance is available in the instance
initializer, this `parent` property can also be accessed. This allows an engine
instance to interrogate its parent, specifically through its `RegistryProxy` and
`ContainerProxy` interfaces.

Alternatively, declarative dependencies can be defined on a limited basis. The
initial API will be limited: an engine can define an array of `services` that it
requires from its parent.

For example, the following engine expects its parent to provide `store` and
`session` services:

```
import Ember from 'ember';

var Engine = Ember.Engine.extend({
  dependencies: {
    services: [
      'store',
      'session'
    ]
  }
});

export default Engine;
```

If necessary, the parent can provide a re-mapping of services from its namespace
to that of the engine via `mount` options in the route map (e.g. to map a
`current-user` service to the engine's `session` service).

When the engine is instantiated, the listed dependencies will be looked up on
the parent and made accessible within the engine.

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

Several incomplete alternatives are discussed in the Motivations section above.

I know of no alternatives being discussed in the Ember community that meet the
same needs as engines; namely, for development _and_ run-time isolation.

# Unresolved questions

## Non-CLI Users

This RFC assumes Ember CLI. I would prefer to prove this out in Ember
CLI before locking down the public APIs/hooks the router exposes for
locating and mounting engines. Once this is done, however, we should
expose and document those hooks so users who cannot use Ember CLI for
whatever reason can still take advantage of composability.

## Declarative dependencies

The initial scope of declarative dependency sharing is limited in scope to
services. Should other types of dependencies be declaratively shareable?
Should addons be the recommended path to share all other dependencies?

## Lazy loading manifests

In order to facilitate lazy loading of engines, we will need to determine a
structure for manifest files that contain an engine's assets. Furthermore, an
application will need to be configurable with URLs for these manifests.

It's likely that an engine's routing map will always be needed at the time of
application deployment. Allowing lazy loading of routing maps would prevent the
formation of any links from a parent application into an engine's routes.

When developed in isolation as addons, engines will have their own sets of
dependencies. These dependencies will be treated like any other addons when
engines are deployed together with an application. However, in order to support
lazy loading, it would be ideal to dedupe dependencies in order to create a lean
and conflict-free asset manifest.

Reference: deduping strategy discussed by @wycats in
[this Google doc](https://docs.google.com/a/tomdale.net/document/d/12CsR-zli5oP2TDWOef_-D28zjmbVD83hU4q9_VTk-9s/edit).

## Namespaced access to engine resources

The concept of namespaced access to engine resources is mentioned above as a
potential goal of a future release of engines. This will require further
discussion to decide how it should work both technically and semantically, and
how it applies to lazy-loaded engines.

If these problems can be resolved, this feature would allow for more flexibility
in parent / engine interactions. Instead of just allowing engines to look up
resources in a parent, the inverse could also be allowed.

For example, if the `authentication` engine contains
`engines/authentication/models/user.js`, a parent application could look up this
same model through a namespace. Perhaps as follows:

```js
container.lookup('authentication@model:user');
```

Other APIs in Ember would need to be extended to support namespaces to
take full advantage of this feature. For example, components that ship
with an engine might be accessed from the primary application like this:

```handlebars
{{authentication@login-form obscure-password=true}}
```
