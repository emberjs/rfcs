- Start Date: 2018-09-12
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Extensible Inspector

## Summary

Unlock experimentation for the Ember Inspector by allowing addons to extend it.

## Motivation

Ember itself has increasingly shifted toward a stable core with experimentation through addons and the Inspector could benefit from a similar philosophical shift. It would move maintenance of debugging functionality away from Inspector maintainers. Instead of Inspector maintainers having to carefully weigh ongoing maintenance costs within the Inspector for each new feature, we could move experiments to addons and co-locate debugging functionality within the same addon that provides the functionality.

I first got excited about this after talking to David Baker and the learning team at EmberConf and David wrote a good explanation at
https://discuss.emberjs.com/t/ember-inspector-call-for-feature-requests-pain-points-and-contributors/15187/20.

### Use cases

#### Data layer
Allow other data layers to implement devtools, for example Redux.

#### Ember addons
Increase the debuggability of Ember addons by providing devtools for them.

Few examples:

- Ember-feature-flags could list all flags registered and allow users to toggle them.
- Ember-mirage could have an editor for its state.
- Ember-animated could show controls for speed of the animations.

#### Devtools only addons
We would also open the possibility to create addons that only contain devtools. This unlocks experimentation for new features for the Inspector without changing the core. It could also be used to provide devtools for the web platform or JS libraries.

#### Application specific devtools
Ember apps could write their own devtools.

- Good for large teams
- Ease new developers into the domain knowledge
- The app is reactive to changes

### Non Ember CLI apps

The addon devtools would only work when there’s an Ember CLI dev server running along with the Ember app. When inspecting an app in production (Non Ember CLI) you would only get what’s currently included in the inspector.

We should focus on enriching the experience for local development while keeping the same functionality for production apps.

The devtools for other framework don’t work on production builds so I wouldn’t be surprised if the inspector would follow that route in the future to allow Ember to be even smaller in production builds.

## Detailed design

To avoid all security risk we can use sandboxed iframes, just like Twiddle and similar tools. The Ember CLI devserver would serve Ember apps under `/_inspector/<addon-name>` (configurable) that would be loaded into an sandboxed iframe and communicate with the inspector with `postMessage`. The inspector is already sending messages from the host site to the Ember app in the inspector. The extensions could register to the events they want to listen to.

## How we teach this

Having a better devtools should make the learning curve better for Ember. The devtools would be installed automatically as you install addons so there’s no need to teach this to the users.

We will need to create documentation for addon authors on how to hook into the inspector and the running Ember app to provide devtools.

## Drawbacks

The main drawback is that this would only work for Ember CLI based apps and not on production builds.

## Alternatives

I first thought about using engines but they do not support dynamically mounting an engine yet (https://github.com/ember-engines/ember-engines/issues/99) and loading 3rd party code into the extension is a security risk.

Instead of getting the devtools through the Ember CLI dev server we could lazily load them from a central repository. The main drawback there are security reasons having it hosted and having to host it somewhere.

## Unresolved questions

Should we share styles for consistency and dark/light theme support or should the apps take care of all styling?
