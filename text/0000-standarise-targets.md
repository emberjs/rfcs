- Start Date: 2017-01-03
- RFC PR: (leave this empty)
- Ember CLI Issue: (leave this empty)

# Summary

This RFC proposes the introdution of a official convention to specify the targets of an
application.

### Motivation

Javascript and the platforms it runs on are moving targets. NodeJS and browsers release new
versions every few weeks. Browsers auto update and each update bings new language features
and APIs.

Developers need an easy way to express intention that abstracts them from the everchanging
landscape of versions and feature matrices, so this RFC proposes to introduce a universal
configuration option to let developers express they intended targets

This configuration should be easily available to addons, but this RFC doesn't impose
any mandatory behaviour on addons. All addons that want to get customize their behaviour
depending on the target platforms will have a single source of truth to get that
information but it's up to them to do it.

The advantage of having a single source of truth for the targets compared to having several
configurations, each one with its own `targets` is mostly for better ergonomics and across-the-board
consitency.

Addons that would benefit from this conventions are `babel-preset-env`, `autoprefixer`,
`stylelint` and `eslint` (vía `eslint-plugin-compat`), but not only those. Even Ember itself could,
once converted into an addon, take advantage of that to avoid polyfilling or even taking
advantage of some DOM API (`node.classList`?) deep in Glimmer's internals.

### Detailed implementation

What seems to be the most popular tool and the state of the art on this is the [browserlist](https://github.com/ai/browserslist)
npm package.

That package is the one behind `babel-preset-env`, `autoprefixer` and others, and uses the data from
[Can I Use](http://caniuse.com/) for knowing the JS, CSS and other APIs available on every browser.

The syntax is also pretty flexible as it allows all sort of complex queries like `Firefox >= 20`,
`>2.5% in CA` (browsers with a market share over 2.5% in Canada) and logical combinations.

Using this syntax would allow us to integrate easily with other tools.

Example usage:

```js
var app = new EmberApp(defaults, {
  targets: ['>2%', 'last 3 iOS versions', 'not ie <= 8']
});
```

This configuration must be made available to addons. It's up to the addons to use it.
I suggest to make it available in `this.project.targets`

```js
module.exports = {
  name: 'ember-data',

  included(app) {
    this._super.included.apply(this, arguments);

    console.log(app.targets); // ['>2%', 'last 3 iOS versions', 'not ie <= 8']
  }
};
```

# How We Teach This

This is a new concept in Ember CLI, so guides will have to be updated to explain this
concept. The good part is that this new concept can help enforcing with tools a task were
traditionally enforced only with peer reviews.

To ease the transition Ember CLI can also, in the absence of a specific value provided by the user,
default to a predefined matrix of browsers that matches the browsers officially supported by the framework.

# Drawbacks

While this RFC standarizes a concept that will open the door to better and more comprehensive tooling,
it makes us choose one syntax (the one used by [browserlist](https://github.com/ai/browserslist)) over
any other perhaps supperior choice that may exist or appear in the future.

# Alternatives

Let every addon that wants to deal with targets to have a `targets`-like option in its configuration
instead of standarizing a single configuration option, which effectively leaves things as they are
right now.

Example:

```
var app = new EmberApp(defaults, {
  'ember-cli-autoprefixer': {
    browers: ...
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

As of today, seems that the database used by [browserlist](https://github.com/ai/browserslist) does
not cover Node.js versions.
With Ember apps running in fastboot this might be a gap in functionality that we have to
fill in a different way, although since this RFC just specifies the syntax, not the tool
that understands it, it is possible that specifying `targets: ['>2%', 'node 4']` becomes
supported and this wouldn't contradict this RFC.