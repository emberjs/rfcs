- Start Date: 2016-12-04
- RFC PR: https://github.com/ember-cli/rfcs/pull/86
- Ember CLI Issue: 

# Summary

Replace PhantomJS with Firefox as the default browser for continuous integration testing.

# Motivation

We want to provide the best possible out-of-the-box continuous integration testing experience for Ember apps. Today that means shipping with configurations for testem and TravisCI. Those configurations use PhantomJS.

But PhantomJS is a weird environment. Users must often fix Phantom-specific browser bugs, which is wasted effort since real users never run your app in Phantom. And "how to debug in Phantom" is an entire extra skill people are forced to learn.

A user-targeted, standards-compliant, modern browser makes a better default choice. Firefox is a good candidate because it's 100% open source, well-supported by Testem on all major operating system, and built-in to TravisCI. Debugging in Firefox has a dramatically nicer learner curve than PhantomJS.

# Detailed design

This is a proposed change to the blueprints for new apps and addons. Existing apps and addons would only be affected when they re-run `ember init` as part of an upgrade and choose to take the updated configuration.

## Changes in testem.js

Replace `PhantomJS` with `Firefox`.

## Changes in travis.yml

Add the following new section to start up a virtual display:

```
before_script:
  - export DISPLAY=:99; sh -e /etc/init.d/xvfb start; sleep 3
```

# How We Teach This

In the guides, replace instructions for installing PhantomJS with instructions for installing Firefox. Since Firefox is a consumer-facing browser with widely-understood installers and behavior, this is one less intimidating thing for newbies to learn.

# Drawbacks

PhantomJS has two primary benefits over other browsers: being headless and being scriptable.

## Headlessness

Firefox is not headless, so it needs to render to a display. That is why the Travis configuration needs xvfb.

## Scriptability

PhantomJS is scriptable, but we don't rely on that functionality anyway. We want cross-browser test suites, so Phantom's scriptability is not particularly useful.

# Alternatives

The default alternative is to do nothing and keep PhantomJS.

Another alternative would be to pick Chrome, since it is a very popular browser. However, Chrome is not 100% open source, which complicates distribution. It's not built into Travis, and the popular methods of installing it there require users to opt into non-container-based images, which are heavier and slower to boot.

Chromium is the fully-open-source parts of Chrome, but like PhantomJS it is an odd duck that's not really well-packaged for end users. It's also not installed by default in Travis.



