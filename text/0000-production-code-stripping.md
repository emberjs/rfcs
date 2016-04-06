- Start Date: 2016-04-06
- RFC PR: (leave this empty)
- ember-cli Issue: (leave this empty)

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

The API documentation will be updated where necessary to indicate that these function calls will be stripped from production builds.

A babel plugin will execute the removal of these function calls based on provided configuration. The plugin will affect the code of the current app or addon only and won't affect code in child or grandchild addons. As this change becomes part of the default ember-cli configuration, addons will adopt the code stripping as they upgrade to newer ember-cli versions.

The plugin configuration will define an array of modules or global functions to remove. Here's an example of what this configuration might look like:

```js
{
  removals: [
    {
      module: 'ember', //eg. import Em from 'ember';
      paths: [
        'assert', //Em.assert will be removed
        'debug',  //Em.debug will be removed
        'a.b.c'   //Em.a.b.c will be removed
      ]
    }, {
      global: 'Ember',
      paths: [
        'deprecate' //Ember.deprecate will be removed
      ]
    }, {
      paths: [
        'console.log' //console.log will be removed
      ]
    }
  ]
}
```

The plugin will support removal of destructured and reassigned invocations of these functions and will support both Babel 5 and 6.

An app or addon can disable the code removal by removing the babel plugin.

# How We Teach This

This change doesn't bring any new functionality. Other than updating the Ember API docs, we don't need to make guide or other documentation changes. At the time of releasing, we may want to point out the possible side effects in a release blog post (see the _Drawbacks_ section below).

If we want to expose the configuration options so that application authors can customize the settings, we can include a new section in the Ember CLI docs.

# Drawbacks

This may introduce an unexpected change in production builds as arguments that have side effects will no longer be executed. For example:

```js
Ember.assert('Some assertion', someSideEffect());
```

Currently, the `someSideEffect` function will be executed in production. When this RFC lands, it won't.

# Alternatives

An Ember addon could provide opt-in function stripping for applications that want it. If this RFC isn't deemed a good default for Ember CLI, that option should be explored.
