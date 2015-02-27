- Start Date: 2015-02-27
- RFC PR:
- Ember Issue:

# Summary

Eliminates Controllers.

# Motivation

As Ember has evolved toward a component-centric model, controllers
have become redundant and vestigal. This unnecessarily burdens
developers with learning an additional concept and forces them to make
unnecessary choices.

The goal of this design is to describe how to eliminate
`Ember.Controller` while maintaining sufficiently rich APIs to do all
the important things controllers do.

# Scope

- Eliminate `Controller` by designing alternative APIs for routing to
  a component, maintaining long-lived non-URL state, and using query
  parameters.

- Simplify the behavior of `Route`'s model hooks. While this is not
  strictly needed to achieve the goal of eliminating controllers, it
  makes sense to address it while we're redesigning the API between
  routes & components anyway.

- Consider how routeable components relate to asynchronous components,
  because they may share concepts or implementation.

- Clarify the distinction between Components and Fragments ("tagless
  components").

# Definitions

 - **routeable component** - a component whose identity and attributes
   are determined from the URL at runtime. Routeable components are
   what go inside `{{outlet}}`s.

 - **asynchronous component** - a component with built-in support for
   asynchronously loading data, with a friendly API for reflecting
   loading and errors states.

Both of the above are conceptual definitions for the purposes of this
design document. They are not necessarily actual classes or concepts
that would need to be taught to new Ember devs.

# Detailed design

## Routing to a component

Our current API for letting a `Route` decide which things to render
and where they should go is fairly rich and complex. We can simplify
it significantly.

### Choosing what component(s) to render

The current `Route.render` method is where a `Route` expresses what
should actually get rendered. It has a fair bit of complexity that we
can deprecate. Today you can say things like:

````js
this.render('post', {
  view: 'special-post',
  controller: 'other-controller' ,
  model: myPost
})
````

Which lets you pick and choose which template, view, controller, and
model to render. There are fairly rich semantics for choosing defaults
for each of those things, and allowing views to choose their own
template, etc.

Instead, we can deprecate the use of the `view`, `controller`, and
`model` arguments, and accept a name which will be used to lookup a
*component*, not a template. We will also accept arbitrary attributes
to set on the component.

````js
this.render('post', attributes)
````

The current semantics of `into` and `outlet` would still be preserved:

````js
this.render('choices', attributes, { into: 'application', outlet: 'sidebar' })
````

We can maintain transitional compatibility by failing to detect a
component and looking for a template instead, using the old behavior
for that case.

Nested child routes have names like 'parent-route/child-route'. Their
corresponding component would also be named
'parent-route/child-route', implying that we must support component
resolution in a non-flat namespace.

Since standalone route templates are going away in favor of
components, the current distinction between the `templates` and
`templates/components` directories becomes unhelpful. We want to put
all of them into just `templates`. Thanks to the introduction of
`<element>` and `<fragment>` (see the
[Element and Fragment RFC](element-and-fragment)), we can make this
transition gracefully.

  - A template in `templates` with an `<element>` or `<fragment>` tag is defining a Component or Fragment.
  - A template in `templates` with neither is a legacy standalone template.
  - A template in `templates/components` with neither is a legacy component.

### Specifying component attributes

We will introduce a new route hook named `attributes`. It receives the
route's positional parameters (the same ones that are currently passed
to the `model` hook), plus any query parameters defined for the route,
and must return a POJO. The POJO's properties may be promises, which
will all be resolved in the same way we currently resolve the `model`
hook (including invoking `loading` actions as needed). The default
`attributes` implementation is:

````js
attributes: function(params, queryParams) {
    return {
        model: this.model(params)
    }
}
````

(Possible variation for this default implementation: any queryParams
that are present get stuffed directly into the attributes as well,
alongside `model`.)

Unlike today's `model` hook, `attributes` gets called *every* time
your route is entered, whether or not `transitionTo` was passed a
preexisting model. This eliminates the need for `beforeModel` and
`afterModel`, because you can implement either behavior from within
your own `attributes` or `model` functions. Therefore, `beforeModel`
and `afterModel` are deprecated.

This also means we deprecate `refresh`, because routes essentially
always refresh. 

### When Routes can call render

Today, `render` is normally called from the `renderTemplate` hook, but
it can also be called from action handlers on the route. Calling
`render` from action handlers -- outside of a route transition --
makes routes unnecessarily stateful and introduces an additional way
to do things that are better off done directly from within components
and their templates. We will therefore deprecate calling `render`
outside of transitions. See Appendix A below for an example of
updating this old behavior.

Today, `renderTemplate` receives the default controller and the
model. But since we don't intend to have controllers anymore, we will
deprecate `renderTemplate` in favor of `renderComponents`, which would
have a default implementation like:

````js
renderComponents: function(attributes) {
    this.render(this.routeName, attributes);
}
````

The `attributes` argument comes from the new `attributes` hook as
described in the previous section. It is resolved with `RSVP.hash`
before passing the result to `renderComponents`.

We also deprecate `setupController`, because there is no longer a
controller to setup -- all state is passed to the component(s) in
`renderComponents` instead.

### Idempotence

Your `attributes` and `renderComponents` hooks are required to be
idempotent (and therefore `model`, which is called from the default
implementation of `attributes`, must also be idempotent if you're
using it with the default `attributes` implementation). This keeps
routes truly stateless and easy to understand. You should not rely on
knowing exactly when or how often a route will be re-invoked, leaving
it up to Ember to optimize.

For cases where you really need to do something different on initial
render, the existing `activate` action will still reliably fire only
when a route is first entered, after Ember invokes your `attributes`
method.

We will add a complementary `update` action that fires after your
`attributes` hook only when the route has not been initially entered.


### Loading and error states

`Routes` will still fire the `loading` and `error` actions as they do
now.

The default handlers for those actions currently look for
correspondingly-named templates. We would change them to look for
correspondingly-named components instead. For example, if before you
had a template named `'people/loading'`, you would instead have a
component named `'people/loading'`.

This is actually a fairly small change, since adding a template is
sufficient to define a component. But it unifies the API, and gives
you the full capability of components for implementing these
substates.

### Sending actions from component up to route

Components are deliberately isolated and don't send arbitrary actions
upward unless you explicitly tell them to. Routeable components
preserve this same behavior.

Therefore, if you want to send an action from a component up to the
Route that rendered it, you should pass an action handler function as
one of your component's attributes.

### Fragments: "Tagless Components"

Today, when you render a template into an outlet, neither the template
nor the outlet introduces extra DOM elements. We want to be able to do
the same thing with routeable components, which means we need to
officially support "tagless components".

But instead of having two flavors of components (tagged and tagless)
with subtly different behavior, we will introduce a new concept:
`Ember.Fragment`. A Fragment is a "tagless component". It still
receives attributes and can maintain internal state, and it renders a
template, but it does not have a top-level element and it may generate
zero, one, or more DOM elements.

`Fragments` have no `element` property. They *do* have a `$` property,
but it represents a DOM range, not a single element.

Everywhere else in this document where I say "component", I really
mean `Component` *or* `Fragment`.

To distinguish `Component` templates from `Fragment` templates, we
will introduce the `<element>` and `<fragment>` keywords, which are
described in a [separate RFC document](element-and-fragment).

## Query Parameters

Today, query parameters live on controllers, which means we need to
find them a new home. Neither Routes nor Components are intended to be
stateful, so we will not bind the values of queryParams directly to
either of those.

queryParams will be *declared and configured* on Routes, with the same
API we currently have on Controllers except for two exceptions. The
first exception is default values -- a query parameter like this:

````js
Ember.Controller.extend({
  queryParams: {
    foo: { as: 'bar' }
  }
  bar: 0
});
````

would look like this on a Route:

````js
  queryParams: {
    foo: { as: 'bar', default: 0 }
  }
````

because the Route itself does not a `bar` property.

The second difference is that we no longer need a `scope`
option. queryParam stickyness is not managed at the Route level, it's
managed inside components using sessions (which have arbitrarily
controllable scopes). More on this below under "Stickyness".

### Data down

When query parameters are changed, their values will arrive in the
queryParams argument to the `attributes` hook. Note that this implies
you will now always be opting in to a "full transition" as described
in the current Guides. This should be ok because each layer is
supposed to be smart about rerendering only things that really need it
-- the change will propagate down through the component hierarchy,
triggering diffing at each step.

### Actions up

To change queryParams, the `Route` will call
`transitionTo`. Components will not know anything about queryParams as
such -- they are just like any other `attributes`. A component that
needs to change a queryParam will use an action handler to communicate
upward to the route as described in the section titled "Sending
actions from component to route".

We can provide a default implementation of a mutable attribute that
can be passed to the component whose setter will trigger the
`transitionTo`. For example:

````js
attributes: (params, queryParams) {
  model: this.model(params),
  filterBy: mut(queryParams, 'filterBy')
}
````

would give the components a mutable attribute that behaves the same as
if the component was invoked directly from a template with:

````handlebars
<the-component model=model filterBy={{mut queryParams.filterBy}}>
````

### Default values

queryParams have default values, which determine when they should
appear in the URL and determine what formatting rules will apply (for
example, a default integer value will cause future values to be parsed
as ints). This behavior can remain the same -- it will just be
configured on Routes instead of Controllers.


### pushState vs replaceState

This can also remain the same. When a Route sets a queryParam it will
use either pushState or replaceState based on the configuration for
that parameter.

### Stickyness

Sticky query params are a specific case of a more general need for
statefulness. Without singleton controllers, this kind of state will
be managed by session services.

Components can use session services to store and retrieve
arbitrarily-scoped state. They can explicitly push that state upward
into the queryParams when they want to.

If you transition into a route that causes a component to render, and
that component has session state for the value of a `mut` attribute,
the component is free to set the `mut` attribute, propagating the
change upward. That change may propagate all the way to the route and
cause a queryParam to be set.

## Idempotent Model Hooks

Today we have `beforeModel`, `model`, and `afterModel` hooks on
`Route`. The distinction between them, and the timing of when they're
called, is complicated by the original goal of avoiding unnecssary
reloading of models. If you `transitionTo('person', 1)` (or directly
set the corresponding URL), all three hooks will fire. But you can
also `transitionTo('person', withThisPersonModel)`, and the `model`
hook will not fire, since a model is already available. `beforeModel`
and `afterModel` exist mostly to deal with this case.

However, we can simplify all of this by moving the problem of avoiding
model reloading into the data layer. `ember-data` is already capable
of sufficiently intelligent caching. 

Therefore, we can collapse the three hooks down to a single `model`
hook *that always fires* regardless of whether the transition
references a model or an id. This gives the user full control over
whether or not they want their data to be served from cache, refreshed
on demand, or a combination of both.

## Asynchronous Components

We already have a nice API for dealing with asynchronous data loading
during route transitions. But we lack a similarly nice API for loading
asynchronous data that does not involve a route transition.

A canonical example of a component that needs this capability is a
file hierarchy browser. Each time the user clicks a node to expand it,
we may need to retrieve additional data, without involving a route
transition.

Today it is relatively straightforward to add this capability to a
component on your own, but it requires every dev to think about the
problem and put together their own solution. We can make a shared
solution that's easier to learn and use.

Once we have asynchronous components, it may be tempting to handle
*all* data loading through them, eliminating traditional `Routes`, and
essentially moving the `model` hook onto `Components`. But we think
there are still strong architectural reasons for having `Routes` as a
separate concept. `Routes` orchestrate transitions, and often have
important work to do before you even know what components need to be
on the page.

The key questions for deciding to load data in a `Route` vs in an
asynchronous `Component` are:

  1. If you're also changing the URL, use a `Route`.
  
  2. If you can't decide exactly where you're going until after
     looking at the (asynchronously-loaded) data, use a `Route`.

  3. Otherwise it's probably fine to use an asynchronous `Component`.


# Drawbacks

There are clearly upgrade costs, but I've attempted to make it
possible to gradually convert a large app.

This is a significant change in mental model that makes the framework
somewhat more opinionated about what Routes are allowed to do. For
example, they can no longer render into outlets at arbitrary times. I
don't think that's a drawback, but it should be considered.

I did not attempt to radically simplify the semantics of the `into`
and `outlet` arguments to Routes's `render` method. I think there's an
argument to be made for doing so, but we would need more design work
to do that.

# Alternatives

It may be possible to achieve some of this RFCs goals with something
like the Engines RFC, by making it possible to "mount" components into
the route hierarchy and then delegate the remainder of the URL for
them to internally route to their children.

# Unresolved questions

- Should the default implementation of the `attributes` hook pass
  queryParams directly as component attributes automatically?

- This proposal makes some assumptions about the semantics of `mut`, a
  concept that has been discussed and sketched out in the Ember 2.0
  RFC but not fully specified.

- I think implementing sticky query params via component sessions and
  mutable attributes is a good idea, but we need to confirm that we
  can make the timing work out such that we don't trigger unnecessary
  double-renders or URL history entries.

# Appendix A: Cross-outlet Communication

The current way to alter another outlet in response to user input is
to call `render` (or its opposite, `disconnectOutlet`) from an action
handler on the Route. For example:

````js
actions: {
  showUploadingStatus: function(status) {
    this.render('upload-status', {
      model: status,
      into: 'application',
      outlet: 'statusbar'
    })
  }
}
````

This RFC deprecates calling `render` outside of route
transitions. Instead, you should render what you need into both
outlets during the initial transition, and link the two components
explicitly via action handlers. Conceptually:

````js
renderComponents: function(attributes) {

  /* This is a hypothetical way to construct a mutable
    attribute. It's intended to reflect the same semantics as as
    the `mut` template helper described in the Ember 2.0 RFC. In
    this case we're just initializing one with an undefined value.
  */
  var uploadStatus = mut();

  // Our main component receives an action in its attributes that
  // it can invoke to set the value of the mutable attribute.
  attributes.setUploadStatus = uploadStatus.setter();
  this.render(this.routeName, attributes);

  // Our upload-status component receives the mutable attribute,
  // which means the component will be invalidated when the
  // attribute changes.
  this.render('upload-status', {
    status: uploadStatus
  }, {
    into: 'application',
    outlet: 'statusbar'
  });
}
````
