- Start Date: 2020-01-09
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/573
- Tracking:

# App Boot Hooks

## Summary

This RFC proposes adding two hooks in the `Application` class in `app.js` that, by default, load
and run initializers and instance-initializers. Here's an eaxmple of what this could look like:

```js
import { loadInitializers, loadInstanceInitializers } from 'ember-load-initializers';
import Application from '@ember/application';
import config from './config/environment';

export default class App extends Application {
    onBoot() {
        loadInitializers(this, config.modulePrefix);
        this.runInitializers();
    }
    onInstanceBoot(appInstance) {
        loadInstanceInitializers(this, config.modulePrefix)
        this.runInstanceInitializers();
    }
}
```

This implementation would be backwards compatible and would give users complete control of
*how* and *when* they want to run their initializers and it would allow the beginner experience to
be progressively enhanced.

## Motivation

- The existing convention of initializer code comes from Ruby on Rails (which has similar
`config/initializers/*.rb` files). This technique works and can be nice for organizing code, but it
is a poor onboarding experience and misses out on a teaching opportunity about how Applications work.
By making it possible to write code as simple as `console.log()` in `app.js`, Ember developers can
be exposed to the entry point of the application much sooner and avoid the complication of additional
directories.
- Currently the order of initializers is not guaranteed without some extra annotations
(`before`/ `after`) that are both error-prone and not obvious. It can be debated whether
two separate initializers *should* care about whether the other one has run or not, it is reasonable
to expect that two unrelated pieces of code will run in the same order every time. This may already
be technically true due to the implemented sorting algorithm, it is not guaranteed by Ember, and
can be messy to grapple with. By using the hooks proposed in the Summary section, Ember developers
can simply write Javascript and expect that it runs the way they intend it to.

## Detailed design

- Add the `onBoot` and `onBootInstance` methods and move the code that runs initializers into these
hooks.
- Expose the hooks and their default implementations into the app blueprint so users can change them
as desired.
- Export a `loadInstanceInitializers` function from the `ember-load-initializers` addon and separate
move the code from `loadInitializers` related to instance initializers into it.

## How we teach this

The Ember Guides currently reference initializers mainly on one page:
[https://guides.emberjs.com/release/applications/initializers/][1].
This page would need to be updated to cover the methods proposed above.

## Drawbacks

Good reasons for separate files per initializer are:

- keeping "concerns separated"
    - this is a valid point for large applications, as it can be useful to encourage team members
    to write new functions rather than writing long `onBoot` functions. It's also reasonable to say
    that choosing the order of initializers should not be a developer concern. However, it is both
    easy to write a messy initializer function in one file, and possible to have tooling that
    prevents a long `onBoot` function. I believe the benefits of `onBoot` outweigh these concerns.
- treeshaking
    - This is also a valid concern, but in a large enough application, it is possible to continue
    using the existing implementation. It's also possible to design the initializer running code
    such that only ONE of the methodologies is possible (functions exported from a directory
    vs `onBoot`).
- container registration
    - Files in the appropriate app directories are automatically registered into the application
    container. Removing code from this directory location and allowing it to sit inline in app.js
    means that initialization functions can no longer be looked up. I have never heard of this
    functionality being used intentionally in production, so I don't think it's a problem.

## Alternatives

- It is already possible to override the `boot` method in `Application` to get this feature,
but it would require re-implementing a large part of the interaction between Application and
ApplicationInstance, it is easily possible for an application to dig itself into a maintenance
nightmare.
- Make initializers more robust and allow them to return promises. This was detailed in [RFC #572],
but the cost of initializers (both in bundle size and maintenance) is already high and it would be
better to drop them entirely. One benefit of initializers that would we would lose is the ability
for addons to inject initialization behavior into their host apps implicitly. While this is useful,
it is also harder to debug this pattern and ultimately makes it harder to maintain Ember apps.

## Unresolved questions

- What constraints, if any, do we want to apply to this free-for-all method?
- Would it be better if the `onBoot` method implemented `loadInitializers` upstream, and then
the user `App` is required to call `super` instead?


[1]: https://guides.emberjs.com/release/applications/initializers/
