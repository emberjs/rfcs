- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Solicit feedback about the support timeframe for Internet Explorer 8 and Internet Explorer 9.

# Motivation

As Ember heads towards version 2.0, it is a good time to evaluate our browser support matrix. Ember follows Semantic Versioning, and we consider browser compatibility to be under the umbrella of those guarantees. In other words, we will continue to support whatever browsers we officially support in Ember 2.0 until Ember 3.0.

Ember 1.x did not have an official browser support matrix, but we would like to correct this for Ember 2.0.

We want to make this decision on the basis of the browsers that our community still needs to support, while weighing that against the costs we bear as a community to support older browsers. This RFC will lay out some of those costs, so we can decide what tradeoff is most appropriate.

Members of the core team maintain many different kinds of apps across many different kinds of companies. Some of us work on applications with small, agile teams, while others work inside of large corporations with many engineers. When this topic came up amongst the team, we discovered that, across all these different companies and Ember apps, no one was still supporting IE8.

Because of this, the core team's impression is that the costs of IE8 support now far exceed the benefits, and we are considering dropping support
for IE8 in Ember 2.0. Before we make the decision, we want to hear from the rest of the community. Supporting IE8 incurs significant cost, both in terms of features and maintenance, and we want the community to help us think through the cost-benefit analysis.

Ember is more than just the framework's code. When people use Ember, they expect to be able to use Ember's tooling, read Ember's documentation, find solutions to problems on Stack Overflow, and read tutorials produced by community members. All of these are shackled to the limitations of IE8, and by dropping support for IE8, people can begin to rely on the improved baseline of ES5.

Below, we outline the costs of continuing to support IE8, so that you can help us make a considered decision.

# Detailed design

## IE8

### Eliminate `get()`

Currently, accessing properties on an Ember object requires using the `.get()` method. By using this abstraction, we have been able to implement several powerful features, such as proxies and computed properties, even on older browsers like IE8 that lack getters and setters.

However, ECMAScript 5, which shipped in **2009**, added support for getters to JavaScript itself, and we would like to use this feature in Ember to eliminate the explicit calls to `get`. Developers new to the framework tell us that having to remember to use `.get()` is a big source of confusion. More seasoned developers get used to it, but moving the Ember object model closer to the pure JavaScript object model is a major goal for Ember 2.x. While many of the features of ES6 classes can be transpiled, getters and setters require engine support, and could not be used if we needed to support IE8.

### More ES6 Features, Today

While much of ES6 can be transpiled correctly to ES3 (the version of JavaScript included with IE8), transpiling ES6 modules and classes requires `defineProperty`.

Continued support for IE8 limits our ability to adopt new ES6 features in the internals of Ember, and to talk about them in our documentation.

One example: In ES6, classes define their methods as non-enumerable properties. Transpiling this to existing browsers is only possible with `defineProperty`, which is not included in IE8. Trying to transpile ES6 classes to work on IE8 would lead to apps exhibiting subtly different behavior that would be painful to debug. IE8 users would discover that the larger Ember ecosystem was incompatible with their apps in hard-to-predict ways, and we think the ecosystem is one of the biggest advantages Ember offers.

In other words, we don't think we can make the full transition to JavaScript classes a first-class part of the Ember experience if we still support IE8. As we did with modules, we would like to move more of our core to JavaScript features in the future, which would be significantly stymied by the lack of `defineProperty` in IE8.

### Remove the jQuery Dependency

For its entire lifetime, Ember has relied on jQuery to smooth the rough edges of browser compatibility when interacting with the DOM. When people think about that dependency, they often assume that we could just replace calls to things like `.attr` with their more verbose DOM counterpart and call it a day.

jQuery does more than just patch over IE8 rough spots; it also serves as the central place for normalizing behavior that can differ significantly across browsers. If we tried to pick-and-choose pieces of jQuery to pull into Ember, we would also be responsible for backporting any changes made to jQuery. We'd rather just rely on jQuery directly; that's what dependencies are for.

The jQuery dependency has helped us with a few cross-browser areas:

* Portable `DOMContentLoaded` (via `jQuery.ready`)
* Support for event delegation across a wide variety of events.
* Attribute and property normalization, which has already been implemented by HTMLBars
* HTML parsing, which has also been implemented by HTMLBars

Of these, proper support for event delegation is the largest remaining reason to rely on jQuery. IE9's support for the capture phase of events makes it simpler to support event delegation properly across all event types without a normalization layer.

### Support More Event Types

Many newly specified events in the web platform (such as the media events) do not bubble, which is a problem for frameworks like Ember that rely on event delegation. However, the capture API, which was added in IE9, is invoked properly for all events, and does not require a normalization layer. Not only would supporting the capture API allow us to drop the jQuery dependency, but it would allow us to properly handle these non-bubbling events. This would allow you to use events like `playing` in your components without having to manually set up event listeners.

### CSS Improvements in Ember 2.x

Today, the main Ember framework does very little to directly help with CSS. We expect that to change in the 2.x series, as we explore ways to help tame the CSS beast.

However, a number of important CSS features landed in IE9: CSS3 selectors, full support for `querySelectorAll`, `getComputedStyle`, `calc()` to name a few. Productively tackling the CSS problem without these features would be like fighting with both hands tied behind our backs, and it may be impossible for us to robustly tackle the problem until Ember 3.0 if we needed to continue to support IE8.

While it may be theoretically possible to implement some form of this feature in IE8, it is likely that the cost of doing so in a backwards-compatible way would significantly add to development time; perhaps so significantly it would be better to wait until we drop support for IE8 than attempt to bolt it on to a browser released half a decade ago.

### Maintenance Costs

While it's very easy to weigh the costs of features that we could not implement at all due to IE8, there is a much more pernicious cost that is harder to see.

Support for IE8 adds costs, sometimes significant, to every new feature we work on. For example, broken support for text nodes in IE8 significantly impeded early work on Glimmer. Every new area of work requires budgeting a significant amount of time for IE8 support.

This is not surprising. When asked many years ago what jQuery could do when IE6 was gone, John Resig replied that we would gain little from dropping IE6, and that the benefits would not come until jQuery could drop IE8, the last version of IE featuring the bugs that made IE6 so difficult to develop for.

Quite often, we will assume that a feature is ready to ship, and only discover subtle issues in IE8 very close to the release once it has been tested. We estimate that support for legacy Internet Explorer slowed down the development of HTMLBars by 2x.

In short, we would be able to implement more features more quickly without the burden of bugs that were first introduced 15 years ago.

## What About IE9?

In the first decade of 2000, browsers were updated very slowly, and every new release took a long time to be supplanted by the next release. As the last version of Internet Explorer supported by Windows XP, IE8 is a relic of this bygone era. In contrast, IE9 usage was quickly supplanted by IE10, and that pattern continues with IE11.

The public trackers have IE9 at a lower share of total usage than IE8, so it might be worth considering dropping them together. Our decision for Ember 2.0 will likely hold until late 2016, so it's worth considering more than just the current moment when making the decision.

While IE9 added support for the ES5 features we need to move into the future for JavaScript, IE10 added support for the last great wave of web features. Here is a sampling:

* Flexbox and Grid Layout
* Offline storage (IndexedDB, File, Blob)
* Web Workers
* Typed Arrays
* Web Sockets
* App Cache
* History API

Several of these features are required for asm.js, and in total, they make the web platform a capable application runtime. While we don't have any immediate plans to take advantage of these web features right now, the best experiments that people are doing today rely on them. By assuming IE10 as the baseline across the entire ecosystem, we would be able to do much more aggressive experimentation on the web platform.

# Drawbacks

Many users have told us that they chose Ember because of the community's commitment to backwards compatibility. When we announced in early 2014 that we would continue to support IE8 for at least another year, other libraries and frameworks had already dropped support. That being said, there will always be organizations using Ember that exist on the tail-end of browser adoption patterns. We risk alienating or upsetting those users by dropping support for a browser that, while on the way out, is not yet completely gone.

However, in many cases, the requirement of IE8 support is driven by non-technical management who do not have a strong sense of the experience of using apps in IE8. In practice, many applications are not rigorously tested in older browsers, and the performance of IE8 is so bad that applications written using any framework perform poorly. Techniques that framework and application developers use to make Chrome fast quite often have pathological characteristics on browsers with a DOM and JavaScript engine written in the 90s.

Still, some people make it work, and dropping IE8 support may prevent those teams from staying with the community as it migrates to Ember 2.0.

# Alternatives

## Drop IE8 Support During 2.x

One alternative we have considered is deprecating IE8 support prior to releasing 2.0, but still maintaining it for a few point releases to give IE8 more time to lose market share.

After discussing with the core team, we believe that this would be a violation of our Semantic Versioning commitment to users. Specifically, we want to avoid a large group of apps getting stuck midway through the 2.x cycle. Version numbers are an important tool for developers, maintainers and ecosystems to communicate compatibility. Tools such as package managers rely on version numbers correctly indicating breaking changes.

We consider browser compatibility to be a feature of Ember, and dropping IE8 support in a minor release would be akin to stripping out any other major feature. While the ecosystem would muddle along in either case, such a move would cause exactly the kind of ecosystem fragmentation that Semantic Versioning is designed to prevent.

If we want to communicate the idea that changing versions comes with a reduction in functionality, we should do that the same way we always do, by incrementing the major version.

## Early 3.0

Another option is to release 3.0 in six months, rather than the nearly two years between Ember 1.0 and Ember 2.0.

Correctly tuning the cadence of major releases is a delicate tradeoff. Semantic Versioning allows us to easily communicate about breaking changes, and some take this as a license to make them frequently. However, a robust ecosystem relies on a certain measure of stability.

We believe that the frustration of breaking changes every six months (or even a year) would outweigh whatever benefits it would provide. Ember's biggest goal is building a shared foundation for our ecosystem to build on, and this requires a careful commitment to stability.

While we could make a "small" breaking release soon after 2.0, breaking changes inherently fragment the ecosystem, and we hope that the years to come bring more stability for add-on authors and tool-makers, not less.

## Bring Your Own Compatibility

Some libraries attempt to thread the needle of IE8 compatibility by asking users to bring their own compatibility libraries. They write the internals of their framework as if IE8 did not exist, and require end users to use polyfills to make the environment look equivalent to newer browsers. For example, React asks users to bring libraries such as `es5-shim`, `es5-sham`, `console-polyfill` and `html5shiv` if they want IE8 support.

Facebook.com supports IE8, and uses React, so there is a path to using React with IE8. This path is partially documented on the React website. This gives us a perfect opportunity to evaluate the impact of this strategy in the real world. We admire the React team's work in this area: support for IE8 is difficult and triaging and fixing IE8 bugs requires diligent effort.

After reviewing the [IE8-compatibility issues filed on React.js tracker](https://github.com/facebook/react/issues?utf8=%E2%9C%93&q=is%3Aissue+is%3Aopen+ie8), we believe there are significant user experience costs to this strategy.

We have spent considerable effort on first-class IE8 support in Ember 1.x, and we feel that users who require IE8 support will have a better experience using Ember 1.14 (with the subset of the ecosystem that supports 1.x) than trying to cobble together a solution that works reliably in a version of Ember with second-class, bring-your-own-compatibility support.

# Unresolved questions

We are relying on the community to help us weigh the above tradeoffs. The more data you can provide about the browser makeup of your customers (especially as it affects revenue), the better we can reason whether now is the time to remove IE8 (and possibly IE9) support.

If you cannot share the information publicly, please email whatever information you consider useful to browserusage@emberjs.com. We will keep it in the strictest of confidence.