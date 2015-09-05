- Start Date: 2015-09-04
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

This feature adds a standard of marking addons with version compatibility to core dependencies such as Ember, Ember-CLI, Ember-Data, etc.

# Motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

The Ember Addon ecosystem is booming, but it is also becoming increasingly crowded with a deluge of information. 
Sites like Ember Addons and Ember Observer attempt to lessen the information overload by categorizing and scoring addons.
However, it is still difficult to identify addons that are outdated or addons that only work past a certain version of Ember, for example.
This RFC details a proposal to standardize tagging of addon compatibility by their authors, so that services such as Ember Addons or Ember Observer
can consume that information to surface to developers in an easy-to-understand format. This could also enable features such as Ember-CLI warning users
when they attempt to `ember install` a package that is listed as incompatible with the core package versions of their application.

# Detailed design

This is accomplished by adding a `compatibility` tag to the `ember-addon` section of an addon's `package.json`. 
This tag should contain a hash of package names that are associated with range strings identifying the compatible versions.
Version ranges should be formatted as outlined by [Node-SemVer](https://github.com/npm/node-semver).
 
Example `package.json`:

```json
{
    "name": "ember-my-addon",
    "version": "0.0.1",
    "description": "My Ember Addon",
    "keywords": [
        "ember-addon"
    ],
    "ember-addon": {
        "configPath": "tests/dummy/config",
        "demoURL": "username.github.io/ember-my-addon",
        "compatibility": {
            "ember": "^1.12.0",
            "ember-cli": "1.13.1 - 1.13.8",
            "ember-data": "1.0.0-beta.18"
        }
    }
}
```

The Ember-CLI blueprint for new addons could possibly include some default compatibility that 
appropriately indicates the compatibility of a the current Ember releases with other versions.
Authors of addons should then update their package.json to reflect the compatibility of their package.

The usage of this feature is not restricted to core Ember libraries, obviously. 
Addon authors could use it to mark other dependencies which they have strict compability with.

# Drawbacks

This is still a manual process depending on other maintainers, and if Ember-CLI adds this to the default blueprint, 
we may see a slew of addons that use the default but never get updated as things become incompatible.

# Alternatives

## What other designs have been considered? 

@kategengler has a similar proposal in an [Ember Observer issue](https://github.com/emberobserver/client/issues/8), 
and @gcollazo is in support of the idea for [Ember Addons](https://github.com/gcollazo/ember-cli-addon-search/issues/64) as well. 
This RFC builds on that concept to provide a standard for the entire Ember ecosystem.

The above issue also discusses leveraging Ember-Try to accomplish this, but the idea was abandoned 
given no way to check if a particular scenario was passing or not. 

## What is the impact of not doing this?

Not doing this continues to leave us with a growing addon ecosystem (Ember Addons shows over 1400 addons so far)
where there is continued confusion over what packages are compatible with certain versions of Ember.  This is particularly
true in the current era of converting from 1.x to 2.x, when it's sometimes difficult to tell if an addon is ready for 2.x yet.

# Unresolved questions

How can Ember-CLI take advantage of this to automate some compatibility validation for developers? (May be a separate RFC)
