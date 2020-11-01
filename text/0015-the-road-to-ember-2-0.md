---
Start Date: 2014-12-03
RFC PR: https://github.com/emberjs/rfcs/pull/15
Ember Issue: This RFC is implemented over many Ember PRs

---

# The Road to Ember 2.0

## Intro

Today, we're announcing our plan for Ember 2.0. While the major version
bump gives us the opportunity to simplify the framework in ways that
require breaking changes, we are designing Ember 2.0 with migration in mind.

This is **not** a big-bang rewrite; we will continue development
on the `master` branch, and roll out changes incrementally on the 1.x
release train. The 2.0.0 release will simply remove features that have
been deprecated between now and then. Our goal is that you can move
your Ember app to 2.0 incrementally, one sprint at a time.

This RFC captures the results of the last two core team face-to-face
meetings, where we discussed community feedback about the future of the
project. While it explains the high-level goals and tries to paint a
picture of how all the pieces fit together, this document will be
updated over time with links to individual RFCs that contain additional
implementation detail.

We plan to flesh out these more-detailed RFCs in the next few weeks,
as the discussion here progresses, before finalizing this plan.

We are announcing Ember 2.0 through our community RFC process in advance
of a release, both so our proposals can be vetted by the community and
so the community can understand the goals and contribute their own ideas
back.

## Motivation

### Stability without Stagnation

Ember is all about identifying common patterns that emerge from the web
development community and rolling them into a complete front-end stack.
This makes it easy to get started on new projects and jump into existing
ones, knowing that you will get a best-of-breed set of tools that the
community will continue to support and improve for years to come.

In the greater JavaScript community, getting the latest and greatest
often means rewriting parts of your apps once a year, as the community
abandons existing solutions in search of improvements. Progress is
important, but so is ending the constant cycle of writing and rewriting
that plagues so many applications.

The Ember community works hard to introduce new ideas with an eye
towards migration. We call this "stability without stagnation", and it's
one of the cornerstones of the Ember philosophy.

Below, we introduce some of the major new features coming in Ember
2.0. Each section includes a transition plan, with details on how
we expect existing apps to migrate to the new API.

When breaking changes are absolutely necessary, we try to make those
changes ones you can apply without too much thought. We call these
"mechanical" refactors. Typically, they'll involve a change to
syntax without changing semantics. These are significantly easier to
adopt than those that require fundamental changes to your application
architecture.

**To further aid in these transitions, we are planning to add a new tab to the
Ember Inspector that will list all deprecations in your application**,
as well as a list of the locations in the source code where the
deprecated code was triggered. This should serve as a convenient
"punch list" for your transitional work.

Every member of the core team works on up-to-date Ember applications,
and we feel the tension between stability and progress acutely. We want
to deliver cutting-edge products, but need to keep shipping, and many
companies that have adopted Ember for their products tell us the
same thing.

### Big Bets

In 2014, we made big bets in two areas, and they've paid off.

The first bet was on open standards: JavaScript modules, promises and
Web Components. We started the year off with globals-based apps,
callbacks and "views", and incrementally (and compatibly) built towards
standards-based solutions as those standards solidified.

The second bet was that the community was as tired as we were of
hand-rolling their own build scripts for each project. We've invested
heavily in Ember CLI, giving us a single tool that unifies the community
and provides a venue for disseminating great ideas.

**In Ember 2.0, Ember CLI and ES6 modules will become first-class parts
of the Ember experience.** We will update the website, guides, documentation,
etc. to teach new users how to build Ember apps with the CLI tools and
using JavaScript's new module syntax.

While globals-based apps will continue to work in 2.0, we may introduce
new features that rely on either Ember CLI or ES6 modules. **You should
begin moving your app to Ember CLI as soon as possible.**

All of the apps maintained by the Ember core team have been migrated to
Ember CLI, and we believe that most teams should be able to make the
transition incrementally.

### Learning from the Community

We're well aware that we don't have a monopoly on good ideas, and we're
always analyzing competing frameworks and libraries to discover great
ideas that we can incorporate.

For example, AngularJS taught us the importance of making early on-ramp
easy, how cool directives/components could be, and how dependency
injection improves testing.

We've been analyzing and discussing React's approach to data flow and
rendering for some time now, and in particular how they make use of a
"virtual DOM" to improve performance.

Ember's view layer is one of the oldest parts of Ember, and was designed
for a world where IE7 and IE8 were dominant. We've spent the better part
of 2014 rethinking the view layer to be more DOM-aware, and the new
codebase (codenamed "HTMLBars") borrows what we think are the
best ideas from React. We cover the specifics below.

React's "virtual DOM" abstraction also allowed them to simplify the
programming model of component-based applications. We really like these
ideas, and the new HTMLBars engine, landing in the next Ember release, lays
the groundwork for adopting the simplified data-flow model.

**In Ember 2.0, we will be adopting a "virtual DOM" and data flow model
that embraces the best ideas from React and simplifies communication
between components.**

Interestingly, we found that well-written Ember applications are already
written with this clear and direct data flow. This change will mostly
make the best patterns more explicit and easier for developers to find
when starting out.

### A Steady Flow of Improvement

Ember 1.0 shipped over a year ago and we have continued to improve the
framework while maintaining backwards-compatibility. We are proud of the
fact that Ember apps tend to track released versions.

You might expect us to do Ember 2.0 work on a separate "2.0" branch,
accumulating features until we ship. We aren't going to do that.

Instead, **we plan to do the vast majority of new work on `master` (behind
feature flags), and land new features in 1.x as they become stable.**

The `2.0.0` release **will simply remove the cruft** that naturally
builds up when maintaining compatibility with old releases.

If we add features that change Ember idioms, we will add clear
deprecation warnings with steps to refactor to new patterns.

Our goal is that, as much as possible, people will be able to boot up
their app on the last `1.x` version, update to the latest set of idioms
by following the deprecation prompts, and have things working on `2.0`.

Because going from the last version of Ember 1.x to Ember 2.0 will be
just another six-week release, there simply won't be much time for us to
make it an incredibly painful upgrade. ;)

## Simplifying Ember Concepts

Ember evolved organically from a view-layer-only framework in 2011 into
the route-driven, complete front-end stack it is today. Along the way,
we've accumulated several concepts that are no longer widely used in idiomatic
Ember apps.

These vestigial concepts make file sizes larger, code more complex, and
make Ember harder to learn.

**Ember 2.0 is about simplification**. This lets us reduce file sizes,
reduce code complexity, and generally make Ember apps easier to pick up
and maintain.

The high-level set of improvements that we have planned are:

* More intuitive attribute bindings
* New HTML syntax for components
* Block parameters for components
* More consistent template scope
* One-way data binding by default, with opt-in to mutable, two-way bindings
* More explicit communication between components, which means less
  implicit communication via two-way bindings
* Routes drive components, instead of controller + template
* Improved actions that are invoked inside components as simple callbacks

In some sections, we provide estimates for when a feature will land.
These are our best-guesses, but because of the rapid-release train model
of Ember, we may be off by a version or two.

However, all features that are slated for "before 2.0" will land before
we cut over to a major new version.

## More Intuitive Attribute Bindings

Today's templating engine is the oldest part of Ember.js. Under the
hood, it generates a string of HTML and then inserts it into the page.

One unfortunate consequence of this architecture is that it is not
possible to intuitively bind values to HTML attributes.

You would expect to be able type something like:

```handlebars
<a href="{{url}}">Click here</a>
```

But instead, in today's Ember, you have to learn about and use the
`bind-attr` helper:

```handlebars
<a {{bind-attr href=url}}>Click here</a>
```

The new HTMLBars template engine makes `bind-attr` a thing of the past,
allowing you to type what you mean. It also makes it possible to express
many attribute-related concepts simply:

```handlebars
<a class="{{active}} app-link" href="{{url}}.html">Click here</a>
```

### Transition Plan

The HTMLBars templating engine is being developed on master, and parts
of it have already landed in Ember 1.8. Doing the work this way means
that the new engine continues to support the old syntax: your existing
templates will continue to work.

The improved attribute syntax has not yet landed, but we expect it to
land before Ember 1.10.

**We do not plan to remove support for existing templating syntax (or
no-longer-necessary helpers like `bind-attr`) in Ember 2.0.**

## More Intuitive Components

In today's Ember, components are represented in your templates as
Handlebars "block helpers".

The most important problem with this approach is that Handlebars-style
components do not work well with attribute bindings or the `action`
helper. In short, a helper that is meant to be used inside an HTML tag
cannot be used inside a call to a component.

Beginning in Ember 1.11, we will support an HTML-based syntax for
components. **The new syntax can be used to invoke existing components,
and new components can be called using the old syntax.**

```handlebars
<my-video src={{movie.url}}></my-video>

<!-- equivalent to -->

{{my-video src=movie.url}}
```

### Transition Plan

The improved component syntax will (we hope) land in Ember 1.11. You can
transition existing uses of `{{component-name}}` to the new syntax
at that time. You will likely benefit by eliminating uses of computed
properties that can now be more tersely expressed using the
interpolation syntax.

**We have no plans to remove support for the old component syntax in
Ember 2.0.**

## Block Parameters

In today's templates, there are two special forms of built-in Handlebars
helpers: `#each post in posts` and `#with post as p`. These allow the
template inside the helper to retain the parent context, but get a piece
of helper-provided information as a named value (such as `post` in the previous examples).

```
{{#with contact.person as p}}
  {{!-- this block of code is still in the parent's scope, but
        the #with helper provided a `p` name with a
        helper-provided value --}}
  <p>{{p.firstName}} {{p.lastName}}</p>

  {{!-- `title` here refers to the outer scope's title --}}
  <p>{{title}}</p>
{{/with}}
```

Today, this capability is hardcoded into the two special forms,
but it can be useful for other kinds of components. For example,
you may have a calendar component (`ui-calendar`) that displays a
specified month.

The `ui-calendar` component may want to allow users to supply a custom
template for each day in the month, but each repetition of the template
will need information about the day it represents (its day of the week,
date number, etc.) in order to render it.

With the new "block parameters" feature, any component will have
access to the same capability as `#each` or `#with`:

```handlebars
<ui-calendar month={{currentMonth}} as |day|>
  <p class="title">{{day.title}}</p>
  <p class="date">{{day.date}}</p>
</ui-calendar>
```

In this case, the `ui-calendar` component iterates over all of days
in `currentMonth`, rendering each instance of the template with
information about which date it should represent.

We also think that this feature will be useful to allow container
components (like tabs or forms) to supply special-case component
definitions as block params. We are still working on the details,
but believe that an approach along these lines could make these
kinds of components simpler and more flexible.

### Transition Plan

Block parameters will hopefully land in 1.12, and at that point the
two special forms for `{{each}}` and `{{with}}` will be deprecated.
You should refactor your templates to use the new block parameters
syntax once it lands, as it is a purely mechanical refactor.

**We have no plans to remove support for the `{{each}}` and `{{with}}`
special forms in Ember 2.0.**

## More Consistent Handlebars Scope

In today's Ember, the `each` and `with` helpers come in two flavors: a
"context-switching" flavor and a "named-parameter" flavor.

```handlebars
{{#each post in posts}}
  {{!-- the context in here is the same as the outside context,
        and `post` references the current iteration --}}
{{/each}}

{{#each posts}}
  {{!-- the context in here has shifted to the individual post.
        the outer context is no longer accessible --}}
{{/each}}
```

This has proven to be one of the more confusing parts of the Ember
templating system. It is also not clear to beginners which to use,
and when they choose the context-shifting form, they lose access to
values in the outer context that may be important.

Because the helper itself offers no clue about the context-shifting
behavior, it is easy (even for more experienced Ember developers)
to get confused when skimming a template about which object a value
refers to.

In Ember 1.10, we will deprecate the context-shifting forms of
`#each` and `#with` in favor of the named-parameter forms.

### Transition Plan

To transition your code to the new syntax, you can change templates
that look like this:

```hbs
{{#each people}}
  <p>{{firstName}} {{lastName}}</p>
  <p>{{address}}</p>
{{/each}}
```

with:

```hbs
{{#each people as |person|}}
  <p>{{person.firstName}} {{person.lastName}}</p>
  <p>{{person.address}}</p>
{{/each}}
```

**We plan to deprecate support for the context-shifting helpers in Ember
1.10 and remove support in Ember 2.0.** This change should be entirely
mechanical.

## One-Way Bindings by Default

After a few years of having written Ember applications, we have observed
that most of the data bindings in the templating engine do not actually
require two-way bindings.

When we designed the original templating layer, we figured that making
all data bindings two-way wasn't very harmful: if you don't set a
two-way binding, it's a de facto one-way binding!

We have since realized (with some help from our friends at React), that
components want to be able to hand out data to their children without
having to be on guard for wayward mutations.

Additionally, communication between components is often most naturally
expressed as events or callbacks. This is possible in Ember, but the
dominance of two-way data bindings often leads people down a path of
using two-way bindings as a communication channel. Experienced Ember
developers don't (usually) make this mistake, but it's an easy one to
make.

When you use the new component syntax, the `{{}}` interpolation syntax
defaults to creating one-way bindings in the components.

```handlebars
<my-video src={{url}}></my-video>
```

In this example, the component's `src` property will be updated whenever
`url` changes, but it will not be allowed to mutate it.

If a template wishes to allow the component to mutate a property, it can
explicitly create a two-way binding using the `mut` helper:

```handlebars
<my-video paused={{mut isPaused}}></my-video>
```

This can help ease the transition to a more event-based style of
programming.

It also eliminates the boilerplate associated with an event-based style
when working with form controls. Instead of copying state out of a
model, listening for callbacks, and updating the model, the `input`
helper can be given an explicit mutable binding.

```handlebars
<input value={{mut firstName}}>
<input value={{mut lastName}}>
```

This is similar to the approach taken by [React.Link][1], but we think
that the use-case of form helpers is sufficiently common to make it
ergonomic.

[1]: http://facebook.github.io/react/docs/two-way-binding-helpers.html

### Transition Plan

The new one-way default is triggered by the use of new component syntax.
This means that component invocations in existing templates will
continue to work without changes.

When transitioning to the new HTML-based syntax, you will likely want to
evaluate whether bindings are actually being mutated, and avoid using
`mut` for values that the component never changes. This will make it
easier for future readers of your template to get an understanding of
what properties might be changed downstream.

To preserve the same semantics during a refactor to the new HTML-based
syntax, you can simply mark all bindings as `mut`.

```handlebars
{{!-- these are semantically equivalent --}}

{{my-video src=movie.url paused=controller.isPaused}}

<my-video src={{mut movie.url}} paused={{mut controller.isPaused}}>
</my-video>
```

While the above example preserves the same mutability semantics, it
should be clear that the video player component should never change the
`url` of the `movie` model.

To make sure you get an exception should this ever happen, simply remove
the `mut`:

```handlebars
<my-video src={{movie.url}} paused={{mut controller.isPaused}}>
</my-video>
```

**We have no plans to remove the old-style component syntax in Ember
2.0, so the semantics of existing component invocations will not
change.**

## Separated Component Parameters

In today's Ember, parameters passed to components as attributes become
properties of the component itself, putting them in the same place as
other internal state.

This can be somewhat confusing, because it may not be obvious to the
reader of a component's JavaScript or template which values are
internal, and which are passed in as part of the public API.

To remind themselves, many Ember users write their components like this:

```js
export default Component.extend({
  /* Public API */

  src: null,
  paused: null,
  title: null,

  /* Internal */
  scrubber: null
})
```

It can also be unclear how to react to a change in the external
properties. It is possible to use observers for this purpose in Ember,
but observers feel low-level and do not coordinate very well with the
rendering process.

To reduce confusion, we plan to move external attributes into a new
`attrs` hash.

If you invoke a component like this:

```handlebars
<my-video src={{movie.url}}></my-video>
```

then the `my-video` component accesses the passed-in `src` attribute as
`this.attrs.src`.

We also plan to provide lifecycle callbacks (modelled after [React's
lifecycle callbacks][react-lifecycle]) for changes to `attrs` that will
integrate with the rendering lifecycle. We plan to supplement the API
with callbacks for changes in individual properties as well.

[react-lifecycle]: http://facebook.github.io/react/docs/component-specs.html#lifecycle-methods

### Transition Plan

In Ember 1.10, we will begin installing provided attributes in the
component's `attrs` hash. If a provided attribute is accessed directly
on the component, a deprecation warning will be issued.

In applications, you should update your component JavaScript and
templates to access provided attributes via the component's `attrs`
property.

**In Ember 2.0, we will stop setting attributes as properties on the
component itself.**

We will also provide a transitional mixin that Ember addons can use that
will make provided attributes available as `attrs.*`. This will allow
add-ons to move to the new location, while maintaining support for older
versions of Ember. We expect people to upgrade to Ember 1.10 relatively
quickly, and do not expect addons to need to maintain support for Ember
1.9 indefinitely.

## Routeable Components

Many people have noticed that controllers in Ember look a lot like
components, but with an arbitrary division of responsibilities. We
agree!

In current versions of Ember, when a route is entered, it builds a
controller, associates a model with it, and hands it off to an
(old-style) view for rendering. The view itself is invisible; you just
write a template with the correct name.

We plan to transition to: when a route is entered, it renders a
**component**, passing along the model as an `attr`. This eliminates a
vestigial use of old-style views, and associates the top-level template
with a regular component.

### Transition Plan

Initially, we will continue to support routing to a controller+template,
so nothing will break. Going forward, routes will route to a component
instead.

In order to do that refactoring, several things will change:

* Instead of referring to model properties directly (or on `this`), you
  will refer to them as `model.propName`.
* Similarly, computed properties that move to your component will need
  to depend on `model.propName` if they are migrated from an
  `ObjectController`.
* In both cases, the short version is that you can no longer rely on the
  proxying behavior of `ObjectController` or `ArrayController`, but you
  can remedy the situation by prefixing `model.` to the property name.
* Unlike controllers, top-level components do not persist across
  navigation. Persistent state should be stored in route objects and
  passed as initial properties to routable components.
* In addition to the asynchronous `model` hook in routes, routes will
  also be able to define a `attrs` hook, which can return additional
  asynchronous data that should be provided to the component.
* Routeable Components should be placed in a "pod" naming convention. For 
  example, the component for the `blog-post` route would be
  `app/blog-post/component.js`.

**We plan to land support for routeable components in Ember 1.12, and
deprecate routeable controllers at the same time. We plan to remove
support for routeable controllers in Ember 2.0.** This will allow
you to move your codebases over to routeable components piecemeal before
making the jump to 2.0.

**We will also provide an optional plugin for Ember 2.0 apps that restores
existing behavior.** This plugin will be included in the Ember automated
test suite to ensure that we do not introduce accidental regressions in
future releases on the 2.x series.

We realize that this is the change has the largest transitional cost of
all the planned features, and we plan to dedicate time to the precise
details in the full RFC on this topic.

## Improving Actions

Today's components can communicate with their parent component through
actions. In particular, the `sendAction` method allows a child component
to invoke a named action on the parent (inside of the `actions` hash).

Part of the reason for this API was a limitation in the original
Handlebars syntax:

```handlebars
{{!-- we can't get too fancy with the value of key-press --}}
{{input key-press="valueChanged"}}
```

In this example, when the `input` component calls
`this.sendAction('key-press')`, it invokes the `valueChanged` action on
its parent component.

With the new HTML syntax for components, we have more flexibility:

```handlebars
<input key-press={{action "valueChanged"}}>
```

This will package up the parent's `valueChanged` action (in the
`actions` hash) as a callback function that is available to the child
component as `this.attrs['key-press']`.

```js
export default Ember.Component.extend({
  keypress: function(event) {
    this.attrs['key-press'](event.target.value);
  }
});
```

The benefit of this approach is twofold:

* Actions are no longer treated specially in the component API. They are
  simply properties packaged up to be called by the child component.
* It is possible to pass an alternative function as the `key-press`,
  reducing the child component's knowledge of what the callback is
  doing. This has testing and abstraction benefits.

### Transition Plan

We will continue to support the `sendAction` API for the forseeable
future in today's Handlebars syntax.

When calling an existing component with new HTMLBars syntax, you do not
need to change your existing `actions` hash. You should change syntax
that looks like this:

```handlebars
{{video-player playing="playingBegins"}}
```

To this:

```handlebars
<video-player playing={{action "playingBegins"}}>
```

The `video-player` component's internal use of `sendAction` will work
with both calling styles.

New components should use `this.attrs.playing()`, but existing components
that want to continue supporting legacy callers should continue to use
`sendAction` for now. The `sendAction` API will seamlessly support both
calling styles, and will be supported for the forseeable future.

```js
// instead of
this.sendAction('progress', value);

// new code can use
this.attrs.progress(value);
```

## Onward

Version 2.0 marks the transformation of Ember from simply an MVC framework
to a complete front-end stack. Between Ember's best-in-class router,
revamped components with virtual DOM, easy-to-use build tools, and a growing
ecosystem that makes taking advantage of additional libraries a breeze, there's
no better way to get started and stay productive developing web apps today.

Hopefully, this plan demonstrates that staying on the cutting-edge can be done
without rewriting your app. There are a huge number of Ember apps in production
today, and we're looking forward to a time in the very near future where they
can start to take advantage these new features.

Expect to see many more RFCs covering these features in depth soon (including
a roadmap for Ember Data 1.0). We look forward to hearing your feedback!

