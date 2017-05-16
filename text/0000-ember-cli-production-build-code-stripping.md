- Start Date: 2016-04-06
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

A number of Ember framework function calls are no-ops in production. Ember CLI should strip these no-op function invocations from production builds by default.

# Motivation

Removing code that isn't required in production results in smaller and faster applications.

# Detailed design

The following framework function calls will be removed from ember-cli production builds by default:

 * [`Ember.assert`](http://emberjs.com/api/#method_assert)
 * [`Ember.debug`](http://emberjs.com/api/#method_debug)
 * [`Ember.deprecate`](http://emberjs.com/api/#method_deprecate)
 * [`Ember.info`](http://emberjs.com/api/#method_info)
 * [`Ember.runInDebug`](http://emberjs.com/api/#method_runInDebug)
 * [`Ember.warn`](http://emberjs.com/api/#method_warn)

The API documentation will be updated where necessary to indicate that these function calls will be stripped from production.

A Babel plugin will execute the removal of these function calls. It will either be:

 * [babel-plugin-filter-imports](https://github.com/ember-cli/babel-plugin-filter-imports) with added support for defining module functions for removal ([spike here](https://github.com/GavinJoyce/babel-plugin-filter-imports/pull/1))
 * a new babel plugin which works similarly to `babel-plugin-filter-imports`

 For consistency, this plugin should also be used in Ember Data and other Ember projects which would benefit from the reduced size.

# How We Teach This

This change doesn't bring any new functionality or visible changes. Other than updating the Ember API docs, we don't need to make guide or other documentation changes.

If we want to expose the configuration options so that application authors can customise the settings, we can include a new section in the Ember CLI docs.

# Drawbacks

TODO:

 * [ ] Are all these functions already no-ops in production? If so, there are no obvious drawbacks

# Alternatives

An Ember addon could provide opt-in function stripping for application who want it. If this RFC isn't deemed a good default for Ember CLI, that option should be explored.

# Unresolved questions

 * [ ] Should we allow the stripping configuration to be overridden in `ember-cli-build.js`? If so, what should the API be?
