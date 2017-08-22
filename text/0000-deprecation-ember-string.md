- Start Date: 2017-07-14
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

This RFC proposes to deprecate the prototype extensions done by `Ember.String` and the `loc` method, and moving `htmlSafe` and `isHTMLSafe` to `@ember/component`.

# Motivation

Much of the public API of Ember was designed and published some time ago, when the client-side landscape looked much different. It was a time without without many utilities and methods that have been introduced to JavaScript since, without the current rich npm ecosystem, and without ES6 modules. On the Ember side, Ember CLI the subsequent addons were still to be introduced. Global mode was the way to go, and extending native prototypes like Ember does for `String`, `Array` and `Function` was a common practice.

With the introduction of [RFC #176](https://github.com/emberjs/rfcs/blob/master/text/0176-javascript-module-api.md), an opportunity to reduce the API that is shipped by default with an Ember application appears. A lot of nice-to-have functionality that was added at that time can now be moved to optional packages and addons, where they can be maintained and evolved without being tied to the core of the framework.

In the specific case of `Ember.String`, our goal is that users that need it will include `@ember/string` in their dependencies, or rely on common utility packages like [`lodash.camelcase`](https://lodash.com/docs/#camelCase).

# Transition Path

Most methods will be extracted to an addon and will show a deprecation message when used from `Ember.String`. This way, if someone wants to use them, they just need to install an addon.

For the last two, similar approach but moving them to the `@ember/component` module.

# How We Teach This

Ember guides' section on _disabling prototype extension_ would need to remove references to these methods. `camelize` is also used in the services tutorial.

Main usage of these functions are in blueprints or connecting to APIs where converting the type of case of keys was needed. It is also used in Ember Data.

For `htmlSafe` and `isHTMLSafe`, the move is easier since the methods are easier related to components than to strings.

In any case, a basic Ember Watson recipe would be easy to provide.

# Drawbacks

A lot of addons that deal with names depend on this behaviour, so they will need to install the addon. Also, Ember Data and some external serializers require these functions.

`htmlSafe` and `isHTMLSafe` would need to change packages, thus the reason to try and provide an Ember Watson recipe.

# Alternatives

Leave things as they are.

# Unresolved questions
