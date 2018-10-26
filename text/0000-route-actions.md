- Start Date: (2018-10-26)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Route Actions

## Summary
This RFC essentially proposes integrating into the core a `route-action` helper that bubbles the
actions all the way up to the routes, just like [ember-route-action-helper](https://github.com/DockYard/ember-route-action-helper).
It's a backwards compatible change.

## Motivation

It has been dubbed that Ember has a steep learning curve. Although (IMHO) this is not
true anymore, this doesn't mean that we can't improve in that area.

Removing the controllers magically and porting their functionality in the routes,
could greatly improve Ember's learning curve.
Albeit this is not possible yet, we could make the controllers less necessary when developing
a new app by embracing this RFC.

At the moment controllers are used for 3 reasons:
  1. decorate/present your model's data
  2. to re-bubble actions to the routes
  3. query params

(1) can easily be avoided using a wrapper component per route template. What is more,
controllers miss vital properties of a regular component class related to DOM. As a result,
a controller becomes quite irrelevant to the DOM and only relevant to the route itself.

(2) can/could be avoided using [ember-route-action-helper](https://github.com/DockYard/ember-route-action-helper)
at the moment. However, the addon is unofficial, has a big warning against using it and is almost unmaintained.

(3) cannot be avoided at the moment and it's the only real reason why one would still need
controllers in an Ember app. However there are 2 things need to be mentioned here:
1. It's way less often to use/need query params than (1) and (2), in general.
2. It well known that query params are problematic in Ember (see [ember-parachute](https://github.com/offirgolan/ember-parachute) and
even @wycats [suggested](https://github.com/emberjs/rfcs/pull/38#issuecomment-355800759) to be removed from the controllers in the future.

It can be easily seen that a `route-action` helper could help minimize the need of controllers
in many apps.

One could ask though: should we really try to minimize the need for controllers per route ?
The answer is yes, anything that can become optional or removed completely without decreasing the
developer's productivity or increasing the framework's complexity should be taken care of.

Another could say: there is an addon for that, we don't just use that?
First of all, there is no way for a newcomer to know this addon. As a result, for newcomers, ember seems to be
more complex than the average/advanced ember developer, as explained above.
Secondly, even if a newcomer could manage to track this addon, the [README](https://github.com/DockYard/ember-route-action-helper/blob/master/README.md) page of that addon even suggests to not use it.
That's even more confusing for a newcomer and possibly for a newbie while it essentially suggests
to make your app a little more complex, spend some more time instead of using the shortcut way.
Note that the addon is very popular even with bit warning message: it has 25,843 weekly downloads when ember-source has
67,954 weekly downloads. This means that 1 out of 3 ember apps use it.

To summarize, a `route-action` helper could provide easier learning curve to newcomers,
by decreasing the complexity of an app while coming closer to routable components.

## Detailed design
Essentially what [ember-route-action-helper](https://github.com/DockYard/ember-route-action-helper) does.
Personally I am not that familiar with Ember's internals but I am pretty sure it's doable and easy to send a PR.
I would happy to work with it.

## How we teach this
It's important to have a different name for actions that will be bubbles up to routes for 2 main reasons:

First, using a different name other than `action`, we make sure that our change is backwards compatible.
Secondly, when used, it makes it clear in the code that an action is intentionally expected to be bubbled up to a route.

Regarding the guides, just a simple example would be sufficient.

## Drawbacks
No drawbacks.


## Alternatives
The alternative is to have the [ember-route-action-helper](https://github.com/DockYard/ember-route-action-helper)
always up to date, by bringing it under [emberjs](https://github.com/emberjs) organization.
But that doesn't really make much sense.

## Unresolved questions
?
