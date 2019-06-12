- Start Date: 2018-10-22
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/391
- Tracking: https://github.com/emberjs/rfc-tracking/issues/14

# Router Helpers

## Summary

This RFC introduces new router helpers that represent a decomposition of functionality of what we commonly use `{{link-to}}` for. Below is a list of the new helpers:

```hbs
{{url-for}}
{{root-url}}
{{is-active}}
{{is-loading}}
{{is-transitioning-in}}
{{is-transitioning-out}}
```

This represents a super set of the functionality provided by [Ember Router Helpers](https://github.com/rwjblue/ember-router-helpers/) which has provided this RFC that confidence that a decomposition is possible.

This RFC does **not** deprecate `{{link-to}}` or `{{query-params}}`. These deprecations will come in the form of a deprecation RFC.

## Motivation

`{{link-to}}` is the primary way for Ember applications to transition from route to route in your application. While this works for a lot of cases there are some use cases that are not well supported or supported at all by the framework. Below is an enumeration of cases that `{{link-to}}` does not address.

### Anchor Tags

We currently do not have a good solution for transitioning solely based on HTML anchors defined in the templating layer. For instance let say you are using [Ember Intl](https://github.com/ember-intl/ember-intl) to do internationalization for your application. Ember Intl uses the [ICU message format](http://userguide.icu-project.org/formatparse/messages) for the actual translation strings and supports having HTML within the string. Now lets say you want to put a link in a translation string and have it work like `{{link-to}}` works. In that case you either have to roll your own solution or use something like [Ember-href-to](https://github.com/intercom/ember-href-to). Another example where this would be useful is that links within markdown produced by addons like [Ember-CLI-Showdown](https://github.com/gcollazo/ember-cli-showdown) would just work. API's like `RouterService#transitionTo` can transition an application using relative URLs and we have an opportunity to leverage this functionality to support this use case.

### Extensibility Of `{{link-to}}`

In 2.11 we moved `LinkComponent` from `private` to `public` largely because there was no other way to modify the behavior of `{{link-to}}` and it had effectively become de-facto public API. That being said, it is less than desirable to `reopen` or `extend` framework objects to gain access to the functionality to create some application specific primitive. For example [Ember Bootsrap](https://github.com/kaliber5/ember-bootstrap/) extends the [`LinkComponent`](https://github.com/kaliber5/ember-bootstrap/blob/master/addon/components/base/bs-dropdown/menu/link-to.js) and then layers more functionality on top of it. Addons would be better served if they had access to more primitive functionality.

### CSS Class Magic

`{{link-to}}` adds some convienent, yet not obvious, classes to the element. These classes are:

- `active`: applied to any `{{link-to}}` that is on the "active" path
- `disabled`: applied depending on the evaluation of `disabled=someBool`
- `loading`: applied if one or more of the models passed are `undefined`
- `ember-transitioning-in`: applied to links that are about to be `active`
- `ember-transitioning-out`: applied to links that are about to be deactivated

The issue with these class names is that they are not declared anywhere in your templated and are provided by the `LinkComponent` as `classNameBindings`. This effectively creates a set of reserved class names that are highly prone to colissions in your typical application.

Furthermore, addons like [ember-cli-active-link-wrapper](https://github.com/alexspeller/ember-cli-active-link-wrapper) and [ember-bootstarp-nav-link](https://github.com/zoltan-nz/ember-bootstrap-nav-link) do a ton of work arounds to get things like the `.active` class to show up on wrapping elements instead of the element directly. This is a great example that shows we are missing some primitives.

### Default Query Param Serialization

Lastly, `{{link-to}}` has very strange behavior when it comes to serializing query params. On a controller you declare the query params for a specific route. These query params can have defaults for them. For example if you have a controller that looks like:

```js
// app/controllers/profile.js
import Controller from '@ember/controller';

export default Controller.extend({
  queryParams: ['someBool'],
  someBool: true,
})
```

and you to link to it like this:

```hbs
{{#link-to 'profile'}}Profile{{/link-to}}
```

In the DOM you will have an `href` on the anchor that gets serializes as:

```html
<a href="/profile?someBool=true" class="active ember-view">Profile</a>
```

Looking at a template you would have no idea that rendering the `{{link-to}}` would result in the query params being serialized. From an implementation point of view, this is problematic as we are forced to `lookup` the `Route` and the associated `Controller` to grab the query params. This can add a non-trivial amount of overhead during rendering, especially if you have many `{{link-to}}`s on a route that link many different parts of your application. As a side-note, this is one of the things  that needs to be dealt with if we are ever to kill controllers.

### Does Not Work With Angle Bracket Invocation

Since angle bracket invocation does not support positional params, `{{link-to}}` has to adapt its public API.

### Costly API

`{{link-to}}` has [a lot of functionality](#kitchen-sink), however this functionality does come at a cost for every instance of `{{link-to}}`. This is not ideal especially if you're just using `{{link-to}}` to generate a url that can be transitioned to. By providing fine grain control of the functionality, applications should see a performance boost.

## Detailed design

Below is a detailed design of all of the template helpers.

### URL Generation Helpers

The following helpers are to be used to construct a valid root-relative URL that will be used by the event dispatcher to perform a transition. These helpers **do not** pass the in memory model, meaning the model hook will always run for the route you are transitioning to.

### `{{url-for}}` Helper

```hbs
{{url-for routeName model queryParams=(hash a=a)}}
```

Or

```hbs
{{url-for routeName models=model queryParams=(hash a=a)}}
```

`{{url-for}}` generates a root-relative URL as a string (which will include the application's rootUrl). When the link is clicked it will cause a transition to occur. See the [Event Dispatcher Changes](#event-dispatcher-changes). It will not serialize the default query params on the controller.

#### Signature Explainer

Using the example above:

_Positional Params_
- **`routeName`** _String_: A fully-qualified name of this route, like `"people.index"`
- **`model`** _...Object|Array|Number|String_: Optionally pass an arbitrary amount of models or identifiers for each dynamic segment to be use for generation.

_Named Params_
- **`models`** _Object|Array|Number|String_: Same as the positional parameter. It is a compiler error if the positional params and the named param is used.
- **`queryParms`** _Object_: Optionally pass key value pairs that will be serialized

_Returns_
- _String_: a root-relative URL as a string (which will include the application's `rootUrl`)

### `{{root-url}}` Helper

`{{root-url}}` simply returns the value from `Application.rootURL`. It can be used to prefix any `href` values you wish to hard code.

```hbs
<a href="{{root-url}}profile">Profile</a>
```

Will result in the following for the default configuration:

```html
<a href="/profile">Profile</a>
```

#### Signature Explainer

`{{root-url}}` does not take any parameters.

### Route State Helpers

The following helpers are all _context dependent_, not global. For instance you might have two copies of `(is-active "posts")` in your app simultaneously where one is `true` and one is `false`, because you're in the middle of an animated transition, or because you're pre-rendering a route that hasn't been entered yet.

### `{{is-active}}` Helper

```hbs
{{is-active 'people.index' model queryParams=(hash a=a)}}
```

or

```hbs
{{is-active 'people.index' models=model queryParams=(hash a=a)}}
```

The arguments to `{{is-active}}` have the same semantics as `{{url-for}}`, however the return value is a boolean. This should provide the same logic that determines whether to put an `active` class on a `{{link-to}}`.

#### Signature Explainer

Using the example above:

_Positional Params_
- **`routeName`** _String_: A fully-qualified name of this route, like `"people.index"`
- **`model`** _...Object|Array|Number|String_: Optionally pass an arbitrary amount of models to use for generation

_Named Params_
- **`models`** _Object|Array|Number|String_: Same as the positional parameter. It is a compiler error if the positional params and the named param is used.
- **`queryParms`** _Object_: Optionally pass key value pairs that will be used to determine if the route is active.

_Returns_
- _Boolean_: Determines if the route is active or not.

### `{{is-loading}}` Helper

```hbs
{{is-loading 'people.index' model queryParams=(hash a=a)}}
```

Or

```hbs
{{is-loading 'people.index' models=model queryParams=(hash a=a)}}
```

The arguments to `{{is-loading}}` have the same semantics as `{{url-for}}` and `{{is-active}}`, however if any of the model(s) passed to it are unresolved e.g. evaluate to `undefined` the helper will return `true`, otherwise the helper will return `false`. This should provide the same logic that determines whether to put an `loading` class on a `{{link-to}}`.

#### Signature Explainer

Using the example above:

_Positional Params_
- **`routeName`** _String_: A fully-qualified name of this route, like `"people.index"`
- **`model`** _...Object|Array|Number|String_: Optionally pass an arbitrary amount of models to use for generation

_Named Params_
- **`models`** _Object|Array|Number|String_: Same as the positional parameter. It is a compiler error if the positional params and the named param is used.
- **`queryParms`** _Object_: Optionally pass key value pairs that will be used to determine if the route is loading.

_Returns_
- _Boolean_: Determines if the route is loading or not.

### `{{is-transitioning-in}}` Helper

```hbs
{{is-transitioning-in 'people.index' model queryParams=(hash a=a)}}
```

Or

```hbs
{{is-transitioning-in 'people.index' models=model queryParams=(hash a=a)}}
```

The arguments to `{{is-transitioning-in}}` have the same semantics as all the other route state helpers, however `{{is-transitioning-in}}` only returns `true` when the route is going from an non-active to an active state. This should provide the same logic that determines whether to put an `ember-transition-in` class on a `{{link-to}}`.

#### Signature Explainer

Using the example above:

_Positional Params_
- **`routeName`** _String_: A fully-qualified name of this route, like `"people.index"`
- **`model`** _...Object|Array|Number|String_: Optionally pass an arbitrary amount of models to use for generation

_Named Params_
- **`models`** _Object|Array|Number|String_: Same as the positional parameter. It is a compiler error if the positional params and the named param is used.
- **`queryParms`** _Object_:  Optionally pass key value pairs that will be used to determine if the route is transitioning in.

_Returns_
- _Boolean_: Determines if the route is transitioning in.

### `{{is-transitioning-out}}` Helper

```hbs
{{is-transitioning-out 'people.index' model queryParams=(hash a=a)}}
```

Or

```hbs
{{is-transitioning-out 'people.index' models=model queryParams=(hash a=a)}}
```

`{{is-transitioning-out}}` is just the inverse of `{{is-transitioning-in}}`.

#### Signature Explainer

Using the example above:

_Positional Params_
- **`routeName`** _String_: A fully-qualified name of this route, like `"people.index"`
- **`model`** _...Object|Array|Number|String_: Optionally pass an arbitrary amount of models to use for generation

_Named Params_
- **`models`** _Object|Array|Number|String_: Same as the positional parameter. It is a compiler error if the positional params and the named param is used.
- **`queryParms`** _Object_:  Optionally pass key value pairs that will be used to determine if the route is transitioning out.

_Returns_
- _Boolean_: Determines if the route is transitioning out.

### Event Dispatcher

In the past, only `HTMLAnchorElement`s that were produced by `{{link-to}}`s would produce a transition when a user clicked on them. This RFC changes to the global `EventDispatcher` to allow for any `HTMLAnchorElement` with a valid root relative `href` to cause a transition. This will allow for us to not only allows us to support use cases like the ones described in the [motivation](#anchor-tags), it makes teaching easier since people who know HTML don't need know an Ember specific API to participate in routing transitions.

#### Route Globs And Route Blacklisting

While the vast majority of the time developers want root relative URLs to cause a transition there are cases where you want root relative urls to cause a normal HTTP navigation. In the router map you can define [wildcard / globbing](https://guides.emberjs.com/release/routing/defining-your-routes/#toc_wildcard--globbing-routes) that makes this problematic as any root relative url can be catched by a wildcard route. To solve this issue this RFC proposes expanding the route options to allow for a black list of urls that are allowed to cause a normal HTTP navigation.


```js
Router.map(function() {
  this.route('not-found', { path: '/*path', blacklist: ['/contact-us', '/order/:order_id'] });
});
```

When an event comes into the `EventDispatcher` we will cross check the blacklist to see if the event should be let through to the browser or if it should be handled internally.

### Transition Attribution

This RFC introduces the notion of an `attribution` to the `Transition`. The `TransitionAttribution` is a read-only object that has 2 fields `event` and `source`.

```ts
interface TransitionAttribution {
  readonly event: Maybe<Event>;
  readonly source: unknown
}

interface Transition {
  readonly attribution: TransitionAttribution;
}
```

On initial render `event` and `source` will be `null`. On subsequent transitions, the `event` will be the DOM event that caused the transition and `element` will be populated with `HTMLElement` that the user interacted with to cause the transition. See [Appendix A](#appendix-a) for example usage. In the event that the transition occurs programmatically through an API like `replaceWith` or `transitionTo` the `event` will be `null` but can be [populated by the caller](#programatic-attribution).

#### Programatic Attribution

In cases where you need to programatically transition with `transitionTo` or `replaceWith` we will allow for you to pass your own `TransitionAttribution`. See Appendix B for an example.

```ts
interface Options {
  queryParams?: Dict<string|number>,
  attribution?: TransitionAttribution;
}

interface Router /* Route, RouterService */ {
  //...
  transitionTo(routeName: string, models?: string|number|object, options?: Options): Transition;
  replaceWith(routeName: string, models?: string|number|object, options?: Options): Transition;
}
```

#### `Transition.attribution` Integrity

The `attribution` in a `Transition` is guaranteed to be carried through the completion of the route transition. This includes `abort`s, `redirect`s and `retry`s of the transition. The `attribution` field is readonlu and the `TransitionAttribution` is readonly and frozen.


## Migration Path

Since this RFC does not deprecate `{{link-to}}` you can continue to use it. That being said `{{link-to}}` has static semantics therefore we can write a codemod using [Ember Template Recast](https://github.com/ember-template-lint/ember-template-recast) to migrate the code. Below are numerous before and after examples of how the codemod would migrate. It's important to note that the behavior of the application [will change](#url-generation-helpers) if you are relying on the passing of the in-memory model. Because of this the codemod would need different levels of conversion.

### Basic `{{link-to}}`

**Before:**

```hbs
{{#link-to 'profile' class="profile"}}Profile{{/link-to}}
{{link-to 'About' 'about'}}
```

**After:**

```hbs
<a href={{url-for 'profile'}} class="profile">Profile</a>
<a href={{url-for 'about'}}>About</a>
```

### With Model `{{link-to}}`

**Before:**

```hbs
{{#link-to 'profile' this.profile class="profile"}}Profile{{/link-to}}
{{link-to 'About' 'about' this.contact}}
```

**After:**

```hbs
<a class="profile" href={{url-for 'profile' this.profile}}>Profile</a>
<a href={{url-for 'about' this.contact}}>About</a>
```

### With Query Params `{{link-to}}`

**Before:**

```hbs
{{#link-to 'post' this.post (query-params order="CHRON")}}{{this.post.name}}{{/link-to}}
```

**After:**

```hbs
<a href={{url-for 'post' this.post queryParms=(hash order="CHRON")}}>{{this.post.name}}</a>
```

One of the trickier parts about this migration is knowing how the autogenerated CSS classes are being used. Because of this, adding the route state helpers must explicitly be turned on in the codemod. For instance if you are making heavy use of the `.active` class, you will be suited best by turning pass the codemod the correct configuration to do a transform like the following:

**Before**

```hbs
{{#link-to 'profile'}}Profile{{/link-to}}
```

**After**

```hbs
<a href={{url-for 'profile'}} class={{if (is-active 'profile') 'active'}}>Profile</a>
```

#### Kitchen Sink

If you were to transform all `{{link-to}}`s verbatim in terms of functionality this would be the result.

**Before**

```hbs
{{#link-to 'profile' model (query-parmas foo=bar) replace=true}}Profile{{/link-to}}
```

**After**

```hbs
<a
  href={{url-for 'profile'
    model
    queryParams=(hash foo=bar)}}
  class="{{if (is-active 'profile' model queryParams=(hash foo=bar)) 'active'}} {{if (is-loading 'profile' model queryParams=(hash foo=bar)) 'loading'}} {{if (is-transitioning-in 'profile' model queryParams=(hash foo=bar)) 'ember-transitioning-in'}} {{if (is-transitioning-out 'profile' model queryParams=(hash foo=bar)) 'ember-transitioning-out'}}">Profile</a>
```

As the kitchen sink example shows, `{{link-to}}` is packed with functionality. While this convienent, it comes with a cost per `{{link-to}}` and is the reason why addons like [Ember-href-to](https://github.com/intercom/ember-href-to) were created. In reality the vast majority of applications only need a subset of this functionality and only in rare cases need things like the transition and loading states.

## How we teach this

In many ways this vastly simplifies Ember's approach to linking within the app. It removes the requirement for a proprietary API and instead embraces the power of URLs.

In the cases where you do need to do more complicated things like pass in memory models to a route, things should feel very similar to `{{link-to}}` as they have the exact same signature. In the case of query param serialization, I believe we are actually aligning a mental model as to how URL generation should work.

## Drawbacks

This RFC expands the surface area of the templating layer by exposing the primitives that make up `{{link-to}}`. This may cause confusion of choosing between using simple basic anchor tags, `{{url-for}}` and `{{link-to}}`, however I believe that each one of the these APIs are solving a real problem that we have in Ember today.

By proxy this may cause people to encapsulate all of these primitives into a single component and thus creating a user-land version of `{{link-to}}`. This could be seen as a framework misstep if the majority of applications end up depending on the addon.

## Alternatives

We could just start deprecating and removing functionality from `{{link-to}}` itself. That being said, it is hard to understand how much of the community is reliant on certain feature of `{{link-to}}`. This also doesn't help with usecases like the i18n and markdown use cases.

## Unresolved questions

TBD?


## Appendix A

```js
// app/utils/tracking.js
const TRACKING_DATA = new WeakMap();
export default TRACKING_DATA;
```

```js
// app/components/track-link.js
import Component from '@ember/component';
import TRACKING_DATA from '../utils/tracking';

export default Component.extend({
  tagName: 'a',
  attributeBindings: ['href'],
  didInsertElement() {
    TRACKING_DATA.set(this.element, this.contextName);
  }
})
```

```js
// app/routes/application.js
import Route from '@ember/routing/route';
import { inject as service } from '@ember/service';
import TRACKING_DATA from '../utils/tracking';

export default Route.extend({
  router: service('router'),
  init() {
    this._super(...arguments);

    this.router.on('routeDidChange', transition => {
      let { source, event } = trasition.attribution;

      let trackingInfo = {
        cause: event.type,
        contextName: null
      };

      if (TRACKING_DATA.has(trasition.source)) {
        trackingInfo.contextName = TRACKING_DATA.get(source);
      }

      ga.send('pageView', {
        from: transition.from ? transition.from.name : 'initial',
        to: transition.to.name,
        attribution: trackingInfo
      });
    });
  }
})
```

```hbs
<h1>Hello {{@name}}!</h1>
<TrackLink href="/profile" @contextName="profile.link">Profile</TrackLink>
<TrackLink href="/about" @contextName="about.link">About</TrackLink>
<TrackLink href="/contact" @contextName="contact.link">Contact</TrackLink>
```

## Appendix B

```js
// app/utils/tracking.js
const TRACKING_DATA = new WeakMap();
export default TRACKING_DATA;
```

```js
// app/routes/profile.js
import Route from '@ember/routing/route';
import { inject as service } from '@ember/service';
import TRACKING_DATA from '../utils/tracking';

export default Route.extend({
  store: service('store'),
  actions: {
    changeName(name) {
      this.model.set('name', name);
    },

    changeAge(age) {
      this.model.set('age', age);
    },

    submit(e) {
      if (isValid(this.model)) {
        let attribution = {
          event: e,
          source: e.target,
        };

        TRACKING_DATA.set(e.target, 'profile.submit');

        this.model.save().then(() => {
          this.transitionTo('profile.success', { attribution });
        }, () => {
          alert('Issue saving... please try again.');
        });
      } else {
        alert('Data is not valid!');
      }
    }
  }
})
```

```hbs
<input onchange={{action 'changeName' value="target.value"}} />
<input onchange={{action 'changeAge' value="target.value"}} />
<button onclick={{action 'submit' this.model}}>Submit</button>
```

```js
// app/routes/application.js
import Route from '@ember/routing/route';
import { inject as service } from '@ember/service';
import TRACKING_DATA from '../utils/tracking';

export default Route.extend({
  router: service('router'),
  init() {
    this._super(...arguments);

    this.router.on('routeDidChange', transition => {
      let { source, event } = trasition.attribution;

      let trackingInfo = {
        cause: event.type,
        contextName: null
      };

      if (TRACKING_DATA.has(trasition.source)) {
        trackingInfo.contextName = TRACKING_DATA.get(source);
      }

      ga.send('pageView', {
        from: transition.from ? transition.from.name : 'initial',
        to: transition.to.name,
        attribution: trackingInfo
      });
    });
  }
})
```

