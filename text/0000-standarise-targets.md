- Start Date: 2017-01-03
- RFC PR: (leave this empty)
- Ember CLI Issue: (leave this empty)

# Summary

This RFC proposes the introduction of a official convention to specify the target browsers
and node versions of an application.

# Motivation

Javascript and the platforms it runs on are moving targets. NodeJS and browsers release new
versions every few weeks. Browsers auto update and each update brings new language features
and APIs.

Developers need an easy way to express intention that abstracts them from the ever-changing
landscape of versions and feature matrices, so this RFC proposes the introduction of a unique
place and syntax to let developers express their intended targets that all addons can use,
instead of having each addon define it in a different way.

This configuration should be easily available to addons, but this RFC doesn't impose
any mandatory behavior on those addons. All addons that want to customize their behavior
depending on the target browsers will have a single source of truth to get that
information but it's up to them how to use it.

The advantage of having a single source of truth for the targets compared to configure this
on a per-addon basis like we do today is mostly for better ergonomics and across-the-board
consistency.

Examples of addons that would benefit from this conventions are `babel-preset-env`, `autoprefixer`,
`stylelint` and `eslint` (vía `eslint-plugin-compat`) and more. Even Ember itself could,
once converted into an addon, take advantage of that to avoid polyfilling or even taking
advantage of some DOM API (`node.classList`?) deep in Glimmer's internals, helping the goal
of Svelte Builds.

# Detailed design

What seems to be the most popular tool and the state of the art on building suport matrices 
for browser targets is the [browserlist](https://github.com/ai/browserslist) npm package.

That package is the one behind `babel-preset-env`, `autoprefixer` and others, and uses the data from
[Can I Use](http://caniuse.com/) for knowing the JS, CSS and other APIs available on every browser.

The syntax of this package is natural but also pretty flexible, allowing complex 
queries like `Firefox >= 20`, `>2.5% in CA` (browsers with a market share over 2.5% in Canada) 
and logical combinations of the previous.

The way this library work is by calculating the minimum common denominator support on a per-feature basis.

Per example, if the support matrix for an app is `['IE11', 'Firefox latest']` and we have a linter
that warns us when we use an unsupported browser API, it would warn us if we try to use
pointer events (supported in IE11 but not in Firefox), would warn us also when using `fetch` (supported
in firefox but not in IE) and would not warn us when using `MutationObserver` because it is supported by both.

This library is very powerful and popular, making relatively easy to integrate with a good amount of
tools that already use it with low effort.

This configuration must be made available to addons but it's up to the addon authors to take advantage
of it.

### Browser support

The configution of target browsers must be placed in a file that allows javascript execution and exports an object
with the configuration. The reason to prefer a javascript file over a JSON one is to allow users to 
dinamically generate different config depending on things like the environment they are building the app in or
any other environment variable.

One possible location for this configuration is the `.ember-cli` file. 
A new dedicated named `/config/targets.js` also seems a good option, similar way how addons use `config/ember-try.js`
to configure the test version matrix.

Ember CLI will require this file when building the app and make the configuration available to addons
in a `this.project.targets` property. 

This `targets` object contains a getter named `browsers` that returns the provided configuration or the default
one if the user didn't provide any.

Example usage:

```js
module.exports = {
  name: 'ember-data',

  included(app) {
    this._super.included.apply(this, arguments);

    console.log(this.project.targets.browsers); // ['>2%', 'last 3 iOS versions', 'not ie <= 8']
  }
};
```

### Node support

In addition to browsers, Ember apps can run in node using ember-fastboot, but this is already covered implicitly 
by the `engines` property in the package.json of the app (or the host app in the case of engines), so no new syntax needed
for defining the support.

This configuration will be available in the same `this.project.targets` object, but on the `node` property.

```js
module.exports = {
  name: 'ember-data',

  included(app) {
    this._super.included.apply(this, arguments);

    console.log(this.project.targets.node); // '>= 4'
  }
};
```

# How We Teach This

This is a new concept in Ember CLI, so guides will have to be updated to explain this
concept. The good part is that this new concept can help enforcing with tools a task were
traditionally enforced only with peer reviews.

To ease the transition Ember CLI can also, in the absence of a specific value provided by the user,
default to a predefined matrix of browsers that matches the browsers officially supported by the framework.

As of today, the supported browser list for Ember.js, according to the platforms we test in saucelabs, is:

`['IE9', 'Chrome current', 'Safari current', 'Firefox current']`

There is no mention to IOS/Android, so this must be validated still.

# Drawbacks

While this RFC standardizes a concept that will open the door to better and more comprehensive tooling,
it makes us choose one syntax (the one used by [browserlist](https://github.com/ai/browserslist)) over
any other perhaps superior choice that may exist or appear in the future.

# Alternatives

Let every addon that wants to deal with targets to have a `targets`-like option in its configuration
instead of standardizing a single configuration option, which effectively leaves things as they are
right now.

Example:

```
var app = new EmberApp(defaults, {
  'ember-cli-autoprefixer': {
    browsers: ...
  },
  'ember-cli-babel': {
    targets: ...
  },
  'ember-cli-eslint': {
    engines: ...
  },
  ...
});
```

# Unresolved questions

The proposed syntax for node only supports a single version of node. Is it reasonable to
make this property an array of versions? P.e. `["4.4", "6", "7"]`
