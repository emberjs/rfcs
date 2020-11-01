---
Start Date: 2017-09-25
RFC PR: (leave this empty)
Ember Issue: (leave this empty)

---

# Summary

Solicit feedback on dropping support for IE9, IE10, and PhantomJS.

# Motivation

As Ember heads towards version 3.0, it is a good time to evaluate our browser support matrix. Ember follows Semantic Versioning, and we consider browser compatibility to be under the umbrella of those guarantees. In other words, we will continue to support whatever browsers we officially support in Ember 3.0 until Ember 4.0.

We want to make this decision on the basis of the browsers that our community still needs to support, while weighing that against the costs we bear as a community to support older browsers. This RFC will lay out some of those costs, so we can decide what tradeoff is most appropriate.
Members of the core team maintain many different kinds of apps across many different kinds of companies. Some of us work on applications with small, agile teams, while others work inside of large corporations with many engineers. When this topic came up amongst the team, we discovered that, across all these different companies and Ember apps, we did not generally support IE9, IE10, and PhantomJS.

Because of this, the core team's impression is that the costs support now far exceed the benefits, and we are considering dropping support for them in Ember 3.0. Before we make the decision, we want to hear from the rest of the community. Supporting IE9, IE10, and PhantomJS incurs significant cost, both in terms of features and maintenance, and we want the community to help us think through the cost-benefit analysis.

Ember is more than just the framework's code. When people use Ember, they expect to be able to use Ember's tooling, read Ember's documentation, find solutions to problems on Stack Overflow, and read tutorials produced by community members. **All of these, including addons that follow Ember’s lead, are shackled to the limitations of these legacy browsers.** By dropping support for them, people can begin to rely on the improved baseline of features.

Some of the features (unavailable in IE9, IE10, or PhantomJS) that addons will be able to freely take advantage of include:

- requestAnimationFrame ([caniuse](http://caniuse.com/#feat=requestanimationframe), [MDN](https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame))
- CSS flexbox ([caniuse](http://caniuse.com/#search=flexbox), [MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Flexible_Box_Layout/Using_CSS_flexible_boxes))
- Websockets ([caniuse](http://caniuse.com/#feat=websockets), [MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API))
- let ([caniuse](http://caniuse.com/#feat=let), [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let))
- const ([caniuse](http://caniuse.com/#feat=const), [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/const))
- TypedArray ([caniuse](http://caniuse.com/#feat=typedarrays), [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray))
- Geolocation API ([caniuse](https://caniuse.com/#search=Geolocation), [MDN](https://developer.mozilla.org/en-US/docs/Web/API/Geolocation))
- Online/offline API ([caniuse](http://caniuse.com/#feat=online-status), [MDN](https://developer.mozilla.org/en-US/docs/Online_and_offline_events))
- XHR advanced features ([caniuse](https://caniuse.com/#feat=xhr2), [specification](https://www.w3.org/TR/2012/WD-XMLHttpRequest-20120117/))
- HTTP2 ([caniuse](http://caniuse.com/#feat=http2), [wikipedia](https://en.wikipedia.org/wiki/HTTP/2))
- Web Workers ([caniuse](http://caniuse.com/#feat=webworkers), [MDN](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers))
- IndexedDB ([caniuse](http://caniuse.com/#feat=indexeddb), [MDN](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)) 
- WebGL ([caniuse](http://caniuse.com/#feat=webgl), [MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API))
- File API ([caniuse](http://caniuse.com/#feat=fileapi), [MDN](https://developer.mozilla.org/en-US/docs/Web/API/File))
- PageTransitionEvent ([caniuse](http://caniuse.com/#feat=page-transition-events), [MDN](https://developer.mozilla.org/en-US/docs/Web/API/PageTransitionEvent))
- SVG filters ([caniuse](http://caniuse.com/#feat=svg-filters), [MDN](https://developer.mozilla.org/en-US/docs/Web/SVG/Tutorial/SVG_Filters_Tutorial))
- MutationObserver ([caniuse](http://caniuse.com/#feat=mutationobserver), [MDN](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver))

Below, we’ve outlined several specific features we’re interested in using to improve the Ember framework itself. We’ve also included some other supporting arguments for this decision.

## Vendor Support

Microsoft dropped most support and maintenance for IE9 and IE10 on 2016-01-16 (IE9 on Vista SP2 [expired in April 2017](http://www.allyncs.com/docs/lifecyclesupport.html)).

With the advent of headless Chrome and Firefox, PhantomJS is now [effectively unmaintained](https://groups.google.com/forum/#!topic/phantomjs/9aI5d-LDuNE). The default testing boilerplate for Ember CLI-generated applications was changed to headless Chrome in Ember CLI 2.15.

## WeakMap, Map, Set

From a framework perspective, being able to rely on native `WeakMap` support will allow us to remove a significant number of fallback paths that are used in browsers without `WeakMap`. Using `WeakMap` results in better developer ergonomics as it allows us to remove many of the random properties that we currently have to assign to an object which makes interacting with your objects in the devtools much less noisy. Minimal support for WeakMap was [introduced in IE11](http://kangax.github.io/compat-table/es6/#test-WeakMap).

## Better ES Class Support

In order to support static class methods (with inheritance) transpilers (e.g. Babel) need to leverage the `Object.setPrototypeOf` / `Object.getPrototypeOf` APIs. Without the ability to rely on `Object.setPrototypeOf` we will not be able to continue iterating slowly towards leveraging ES classes as a replacement for the custom object model functionality that we have known and loved for so many years. Specifically, there is no replacement / capability to support proper inheritance with `.reopenClass`. There are several lower-fidelity hacks you might opt into, but none that we think satisfy the needs of the Ember community.

Generally this means IE11 is the oldest browser we can reliably transpile ES classes for reliably.

## Typed Arrays

Typed arrays are not currently used in Ember, but experimentation is underway deep in the internals of Glimmer VM to be able to further reduce template size *and* the costs associated with expanding the wire format (currently a JSON structure) into a runnable program. Leveraging typed arrays would allow Ember and Glimmer apps to completely avoid the wire format to opcode compilation that currently happens before initial render. It also significantly reduces the resulting memory footprint for the same runnable program.

## DOM API Improvements

Although IE9 introduced JavaScript engine with support for much of ES5, it was not until IE10 that the browser began to support much of what developers consider modern web platform APIs. Littered throughout the Ember and Glimmer VM codebase are [many](https://github.com/glimmerjs/glimmer-vm/blob/1759c16defc546b034b97e37141187652ed93859/packages/%40glimmer/runtime/lib/dom/props.ts#L54) [examples](https://github.com/glimmerjs/glimmer-vm/blob/9ecc88504c81469ba20dba3ed3f37d373a998355/packages/%40glimmer/test-helpers/lib/helpers.ts#L170) [of](https://github.com/glimmerjs/glimmer-vm/blob/bfed16af6a5ecce4fbe9f27783245fe0f8b03480/build/broccoli/transpile-to-es5.js#L25) IE9 workarounds (and [PhantomJS workarounds](https://github.com/glimmerjs/glimmer-vm/blob/1759c16defc546b034b97e37141187652ed93859/packages/%40glimmer/runtime/lib/dom/props.ts#L49), in fact). We’ve worked hard to make these fixes free at runtime for modern browsers, but some cost is unavoidable.

PhantomJS in particular is a weird environment. Users must often fix Phantom-specific browser bugs, which is wasted effort since real users never run your app in Phantom. And "how to debug in Phantom" is an entire extra skill people are forced to learn. Testing your app in PhantomJS is generally a form of “testing theater”, since it fails to execute your code in a realistic environment.

## `requestAnimationFrame`

IE10 introduced support for `requestAnimationFrame`, an efficient way to schedule work in the browser environment. We’re interested in using this API to explore incremental rendering strategies, and as a way to improve Ember’s coordination with the browser when native promises are used in application code.

# Detailed Design

When using Ember applications in IE9, IE10, or PhantomJS, Ember will cause an appropriate deprecation to be issued. The deprecation will be “until 3.0” and will reference an entry in the deprecation guide. The guide entry will describe For example:

> Using Ember.js in IE9, IE10, or PhantomJS is deprecated and will be unsupported in Ember.js 3.0. We recommend using Ember’s 2.x LTS releases if your applications must support those browsers.
> 
> PhantomJS is often used for continuous integration testing. We strongly suggest adopting headless Chrome or Firefox to run CI tests.

# Drawbacks

Many users have told us that they chose Ember because of the community's commitment to backwards compatibility. There will always be organizations using Ember that exist on the tail-end of browser adoption patterns. We risk alienating or upsetting those users by dropping support for a browser that, while on the way out, is not yet completely gone.

However, in many cases, the requirement for supporting these legacy browsers is driven by non-technical management who do not have a strong sense of the experience of using apps in IE9/IE10. In practice, many applications are not rigorously tested in older browsers, and the performance is so bad that applications written using any framework perform poorly. Techniques that framework and application developers use to make Chrome fast quite often have pathological characteristics on browsers with legacy DOM and JavaScript engines.

Still, some people make it work, and dropping support may prevent those teams from staying with the community as it migrates to Ember 3.0.

As a mitigation for these concerns, the final release of Ember 2.x will itself be made an LTS release. This will ensure a 2.x platform supporting IE9+ with critical bugfix for roughly 8 months following the 3.0 release and security fixes for roughly 14 months after 3.0 release.

# Alternatives

## Bring Your Own Compatibility

Some libraries attempt to thread the needle of compatibility by asking users to bring their own compatibility libraries. They write the internals of their framework as if these older browsers did not exist, and require end users to use polyfills to make the environment look equivalent to newer browsers.

We have spent considerable effort on first-class support in Ember 2.x, and we feel that users who require IE9 and IE10 support will have a better experience using Ember 2.x. (with the subset of the ecosystem that supports 2.x) than trying to cobble together a solution that works reliably in a version of Ember with second-class, bring-your-own-compatibility support.
