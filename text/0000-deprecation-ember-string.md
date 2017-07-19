- Start Date: 2017-07-14
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

This PR proposes the deprecation of `Ember.String`. The following methods will be moved to an addon:

- `camelize`
- `capitalize`
- `classify`
- `dasherize`
- `decamelize`
- `fmt`
- `loc`
- `underscore`
- `w`

The remaining methods (`htmlSafe` and `isHTMLSafe`) will be moved to `@ember/component`.

In both cases, the String prototype won't be extended anymore.

# Motivation

`Ember.String` was introduced long ago, even before 1.0 was released. It was a time without the current ecosystem of addons. There were no ES6 modules, no Ember CLI and no Ember addons. Global mode was the way to go and prototypes of classes like `String`, `Array` and `Function` were being extended.

A lot of nice-to-have functionality was added at that time but now Ember is lifting weight. These functions belong in an addon, where it can be maintained and evolve without being tied to the core of the framework.

Also, this would let people swap the function for similar implementations but different behaviour in edge cases.

Once most of the methods are moved to an addon and `String` is no longer being extended, makes more sense to move `htmlSafe` and `isHTMLSafe` methods to the component layer.


# Transition Path

Most methods will be extracted to an addon and will show a deprecation message when used from `Ember.String`. This way, if someone wants to use them, they just need to install an addon.

For the last two, similar approach but moving them to the `@ember/component` module.

# How We Teach This

Ember guides' section on _disabling prototype extension_ would need to remove references to these methods. `camelize` is also used in the services tutorial.

Main usage of these functions are in blueprints or connecting to APIs where converting the type of case of keys was needed. It is also used in Ember Data.

Furthermore, when people have moved to the new shims, moving to an addon would be as easy as to change the import path.

For `htmlSafe` and `isHTMLSafe`, the move is easier since the methods are easier related to components than to strings.

In any case, a basic Ember Watson recipe would be easy to provide.

# Drawbacks

A lot of addons that deal with names depend on this behaviour, so they will need to install the addon. Also, Ember Data and some external serializers require these functions.

`htmlSafe` and `isHTMLSafe` would need to change packages as well, thus the reason to try and provide an Ember Watson recipe.

# Alternatives

Leave things as they are.

# Unresolved questions
