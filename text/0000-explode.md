- Start Date: 2017-12-07
- RFC PR: 
- Ember Issue: 

# Summary

This RFC proposes a way to begin immediately splitting Ember into a family of separate NPM packages without breaking ecosystem compatibility.

# Motivation

This proposal

 - enables people working on small, focused apps to start with a minimal, small core of Ember and only add more features as they are needed
 
 - allows us to more aggressively isolate legacy features by making them pay-as-you-go
 
 - creates an upgrade incentive for apps and addons that is "more carrot, less stick".
 
# Detailed design

## @ember/core Package

As of Ember 3.0, the only supported way to depend on Ember is to depend on the `ember-source` npm package. This proposal does not change the meaning of `ember-source`. It will continue to be all-of-Ember in the way people already understand it, and features can only be removed from `ember-source` at major releases after an appropriate deprecation cycle.

We propose adding an *additional* supported way to depend on Ember. You can choose to run `ember explode`, and we will remove `ember-source` from your dependencies and replace it with `@ember/core` *plus the current complete set of already-extracted Ember packages*. `ember explode` is intended to not change the set of Ember features in your app. It just splats them out into a form that comes from separate packages.

Critically, `@ember/core` includes an extra rule in its semver policy: the only supported way to upgrade to a newer `@ember/core` is to run the `ember-cli-update` command. The update command will be aware of any additional packages have been split off from `@ember/core` since your last version, and it will add them automatically to your app.

This special upgrade requirement gives us the freedom to split off more packages *in any release*, without breaking anybody's app.

## Inter-package dependencies

If one of the newly-split-off Ember packages depends on another, it will say so in its NPM peerDependencies. We will use exact-version constraints, which is effectively how things already work today.

NPM has historically been loose about peerDependencies, so they are often ignored by developers. We propose that ember-cli should hard error for missing peerDependencies to avoid this problem (most people who think they don't really need to clean up peerDependency warnings are simply mistaken, and have latent bugs).

## Addon dependencies on Ember packages

Right now, every addon in the ecosystem is allowed to implicitly depend on any feature in Ember. You can't know *a priori* which features are safe to remove if you're using any given addon.

Therefore, we propose new metadata in the `ember-addon` section of `package.json`:

```js
  "ember-addon": {
    "required-core-features": "3.1.0",
    "required-additional-features": ["@ember/routing", "@ember/prototype-etxensions", "@ember/string"]
  }
```

`required-core-features` records the latest version of `@ember/core` that the addon has been tested against. It causes your addon to depend on all the Ember packages that were still bundled inside that `@ember/core` version. 

`required-additional-features` lets the addon express dependency on features that have already been split from `@ember/core` as of the base features version.


### Why required-core-features instead of peerDependencies?

`required-core-features` says something different than a `peerDependency` constraint.

We want addons to be as aggressive as possible in bumping their `required-core-features`, because that creates the maximum opportunity for apps to pick their packages *a la carte*.

Simultaneously, we want addons to support the widest practical range of Ember versions (which would be their `peerDependencies` constraint on Ember, if they choose to have one), because that makes everybody's upgrades easier. 

For example, consider an addon with:

```js
  "ember-addon": {
    "required-core-features": "3.1.0",
    "required-additional-features": ["@ember/routing"]
  }
```

If you use this addon in an app with `ember-source` 2.18.0, there's no problems, the metadata isn't affecting anything.

If you use this addon in an app with `@ember/core` 3.6.0, there's still no problems, but we will be able to warn the user that they need to install `@ember/routing` plus whatever future packages happen to get split out of `@ember/core` between 3.1 and 3.6.

In contrast, if there was a peerDependency on `@ember/core` we would break usage with `ember-source`, and *vice versa*. And if there was a peerDependency on `@ember/routing` it would break usage with any Ember version earlier than the one that split out that package.

The "pure NPM" way of solving this problem is to release new versions of your addon to support new versions of Ember, but this creates a much more fragmented ecosystem than we want. We are careful not to break things very often, such that many addons really don't need to be released just to keep working with newer Ember releases.


## Local Dependency Overrides

We have already proposed making missing dependencies a hard build error. This is important so that weird and frustrating bugs don't crop up unexpectedly. But it needs to be possible to get oneself unstuck without being blocked by third-party code.

Therefore, we propose a `config/dependency-overrides.js` file that allows you to make declarations on behalf of any addon:


```js
/* 
  If you need to edit this file, please file PRs to share these 
  changes with the addon's authors! 
*/
module.exports = {
  "liquid-fire@^0.29.0": {
    "required-core-features": "3.4.0",
    "required-additional-features": [
      "@ember/component"
    ]
  }
};
```

The point of this file is purely to suppress the build errors you would otherwise receive. 

Each override is scoped to exactly the package *and* version constraint that you're using. If you change the version of the addon that you're using, the override will no longer apply. This is deliberate -- there's no way to safely know what overrides to apply to a new version of the addon.

## Package Dependency Example Scenario

*All examples in this section may use fictional package names, version numbers, and relationships. For illustration purposes only.*

### Scenario: Depending on an addon with insufficient required-base-features

1. Your app depends on `ember-source` 3.2.0. 

2. You decide to run `ember explode` to try out this brave new world. It changes your package.json like this:

    ```diff
    <     "ember-source": "3.2.0",
    ---
    >     "@ember/core": "3.2.0",
    >     "@ember/partial": "3.2.0",
    >     "@ember/prototype-extensions": "3.2.0",
    ```

3. Your app is expected to work exactly the same at this point.

4. You decide "I'm not using partials in my app", so you remove `@ember/partial`.

5. `ember serve` gives you an error report that includes a message like:

    ```
    Addon liquid-fire 0.29.0 may depend on `@ember/partial` to function correctly. You should either add `@ember/partial` to your app or update to a version of liquid-fire that declares `required-base-features` >= 3.1.0. If you want to test whether it actually works, you can override this warning, see https://...
    ```

    In the above example, the message said we need to get liquid-fire to declare a required-base-features of at least 3.1.0 because (in this hypothetical example), that is the first version of `@ember/core` that doesn't include the features in `@ember/partial`.
    
6. You update liquid-fire and your app is running happily now without the feature. 

### Scenario: Addon author declaring dependencies

1. Ember 3.2.0 is released, and it splits out `@ember/partial` and `@ember/prototype-extensions` as their own packages.

2. You want to update your `mega-button` addon to take advantage of this new capability. You test your addon against `@ember/core` 3.2.0 and get a test failure:

    ```
    You're trying to use a feature that was moved out of `@ember/core` and into `@ember/partial`
    ```

    In this case, you got a pleasant error message because we were able to leave a development-mode-only assertion in place of the removed feature. It looks like your mega-button is actually using partials after all.
    
3. You update your addon's test scenario to use both `@ember/core` and `@ember/partial` 3.2.0. Now tests pass. This confirms that those are the only packages you depend on.
    
4. You update your package.json to include:

    ```
    "ember-addon": { 
      ...
      "required-base-features": "3.2.0"
      ...
    }
    ```
    
    because that is the latest version you have tested. And you also add:
    
    ```
    "ember-addon": {
       ...
       "required-additional-features": ["@ember/partial"]
       ...
    }
    ```
    
    because that is the only package you discovered you need that is outside of `@ember/core`.
    
5. The final result of your work is that apps who use your addon are now free to remove `@ember/prototype-extensions`, but they cannot remove `@ember/partial`.

## How this relates to Svelte Builds

"Svelte builds" is the idea of doing feature flagging in reverse, so that code that's deprecated but not-yet-removed can be stripped out by users who don't depend on it.

This proposal is designed to enable the use of svelte builds *as an implementation detail* that apps and addons don't need to worry about. 

A feature that we want to svelte away can be "moved" into a new package, and users who don't want the feature can simply drop that package. I'm using scare quotes around "move" because the new package itself can be as simple as a private build-time flag that alters which features get stripped from `@ember/core`. This strategy has already worked well in the past for other legacy support packages.

## How this relates to deprecations

The existing deprecation system is complementary to this proposal, and I'm not proposing any changes to deprecations.

We can still deprecate features and leave them in `@ember/core`. But we can also choose to quarantine deprecated features into newly-split-out packages, making them easier for apps to drop immediately.

The two systems have complementary strengths and weaknesses. Only deprecations can permanently remove code that we don't want to support anymore. But package splitting can allow apps to stop shipping deprecated code as soon as they are ready, without waiting for everyone else.

## How this relates to JS Module API

[RFC 176](https://github.com/emberjs/rfcs/blob/master/text/0176-javascript-module-api.md) defined a bunch of packages that are, today, not real NPM packages. This proposal gives us the infrastructure to begin making them into real packages.

However, we are also free to create additional packages that are not in that RFC, particularly in cases where we want to isolate a particular feature or behavior that is on the road to deprecation and removal. Creating a new package requires an RFC (possibly a deprecation RFC).

## Development-mode stubs

It will sometimes be convenient to leave development-mode-only stubs in place of removed code so that we can provide helpful guidance. This should be pursued whenever possible, and it can be done automatically wherever we are doing block-level code stripping.


## New App Blueprint

We should create a new app blueprint that allows people to start with just `@ember/core`. There is no plan at this time to replace the default app blueprint -- this would be a new choice for people who want to start with a very barebones Ember, similar to what you'd get with GlimmerJS.

    ember new <app-name> --blueprint @ember/core

# How We Teach This

The most important message we need to teach app developers is: use `ember-cli-update` whenever you're changing your Ember version. As long as we can spread that message, we can provide direct guidance the rest of the way. For example, when we reach a sufficient level of confidence in the `@ember/core` packaging, we can begin making `ember-cli-update` offer to automatically explode `ember-source` into `@ember/core` *et al*. This would be a simple code mod that should not alter any app semantics.

As people begin to experiment with removing Ember packages, our best teaching opportunities are very clear and helpful feedback from `ember-cli` whenever they have a dependency issue, as illustrated in some of the example messages in this document.

The mental model for "which Ember packages am I using?" should ultimately reduce to simple rules:

 - in Javascript, you can see directly from your ES imports which packages you are depending on.
 - in templates you will often be able to see what package a component or helper comes from thanks to [RFC 143](https://github.com/emberjs/rfcs/blob/master/text/0143-module-unification.md), which adds namespacing.
 - Ember-provided components and helpers don't necessarily require namespaces, but we can offer precise guidance if you use one that isn't installed, like "No {{link-to}} component found. You may need to install @ember/routing."

We don't need to make any immediate changes to the guides, because the default behavior of Ember isn't changing. And even if people copy code examples out of more "barebones" Ember apps, all those examples will work in a default (complete, `ember-source`-based) app.

As the barebones app experiences (described above under New App Blueprint) matures, we should develop a separate guide designed around how to build apps with that starting point. 

# Drawbacks

The biggest drawback of Ember-as-many-packages is that NPM's peerDependency support is relatively weak. Simply saying `npm install @ember/foo` won't automatically take peerDependencies into account to choose the precise version of `@ember/foo` that matches your other packages. We can easily tell you when your peerDeps are wrong, but we can't always easily fix them for you automatically, unless we start laying specialized tools on top of NPM.

In the case of code we want to remove, splitting it into separate packages doesn't eliminate our need to support it (at least until the next major release lets it be truly removed). So while this proposal lets us "go faster" in some senses, it doesn't let us reduce support any faster.

# Alternatives

We could continue to ship Ember as a single package, and do our own Ember-specific built-time shenanigans to decide how much of it to include in the app's build. This greatly simplifies the management of user's package.json files, but it is increasingly out-of-step with wider NPM-based Javascript toolchains.

We could choose to only split out new packages at major releases. This would eliminate the need for mandatory `ember-cli-update`, at the cost of either shipping many more breaking releases or waiting much longer to benefit from package splitting. I think `ember-cli-update` is the superior option because it gives us a way to refactor Ember without doing breaking changes.
