- Start Date: 2017-06-18
- RFC PR: (leave this empty)

# Summary

This RFC proposes to add a new API to allow addons to register a custom transformation. This transformation can then be used by other addons when calling `app.import` with `using` API.

# Motivation

Addons or apps may want import browser only compatible libraries using `app.import` via bower or npm. These libraries should not be running in Node.

When FastBoot was doing two builds (to generate different assets for browser and Node environment), addon or apps often conditionally imported these libraries relying on the value of `process.env.EMBER_CLI_FASTBOOT`. With the new scheme of the build where only additional Node assets are built, this enviornment is no longer exposed.

In order to expose better semantics to allow apps and addon authors to easily import these libraries without much overhead (see issue [here](https://github.com/ember-fastboot/ember-cli-fastboot/issues/413)), we need to have these libraries wrapped with an FastBoot check. This can be achieved by extending the `using` API of `app.import`. FastBoot addon would like to register a custom transformation that other FastBoot compatible addons may chose to use in a declarative API.

# Detailed design

Today, `ember-cli` supports transforming anonymous AMD modules imported via `app.import` into named AMD modules:

```js
app.import('/path/to/module.js', {
  using: [
    { transformation: 'amd', as: 'some-dep' }
  ]
});
```

The `amd` transform is hardcoded in `ember-cli`. However, it is not possible for addon authors to provide any additional transformation that other addons can use when importing third-party modules. Addons like, FastBoot would like to provide custom transformation for other addons to use so that they can wrap their third party libraries in Node environments.

In order to do this, we would like to expose an API that allows addons to register a custom transformation. This API will be an advanced API and will only be used by addons that want to provide custom transformation. Other addons can chose to use that custom transformation using its name.

The API to register a custom transformation in `ember-cli` will be defined in `index.js` of the addon and will be an advanced API:

```js
addCustomTransform() {
  return {
    'fastboot-shim': function(tree, options) {

      return stew.map(tree, function(content, relativePath) {
        return `if (typeof FastBoot === 'undefined) { ${content} }`;
      });
    }
  }
}
```

`addCustomTransform` returns a map of the name of the transform and a callback function that will be run on every module that uses the transform. The callback function takes the `tree` as broccoli tree contain all the files that want to run this transform and `options` map (optional) that contains the additional key value pairs that a consumer transformer provides. The later argument would be used by transformations like `amd` (explained below).

With this, we also should move the hard coded `amd` transform into an in-repo addon in `ember-cli`. This would allow other addons that define their own transformation to also control the order of their transformation (using `before` or `after` hooks of addon initialization). The registeration of `amd` transform would be:

```js
addCustomTransform() {
  return {
    'amd': function(tree, options) {

      return stew.map(tree, function(content, relativePath) {
        const name = options[relativePath].using;
        if (name) {
          return [
            '(function(define){\n',
            content,
            '\n})((function(){ function newDefine(){ var args = Array.prototype.slice.call(arguments); args.unshift("',
            name,
            '"); return define.apply(null, args); }; newDefine.amd = true; return newDefine; })());',
          ].join('');
        } else {
          return content;
        }
      });
    }
  }
}
```

As seen above, `options` contains the optional AMD module ID that the consumer of `amd` transform can provide. If registered transforms want to depend on any other user provided values, those can easily be available during the transforms.

When the addons are initialized, we will check if `addCustomTransform` is defined and store these callbacks and transform names in an array.

Now, if addon authors would like to use these transforms when importing libraries, they would simply do the following:

```js
app.import('/path/to/module.js', {
  using: [
    {  transformation: 'fastboot-shim' },
    { transformation: 'amd', as: 'some-dep' }
  ]
});
```

As seen above, an addon author could provide the list of transformations to run and `ember-cli` would run them in the order of when the transformations were registered.
Internally, for every transform we will maintain an array of file paths that need to run this transform. When the transformations need to run, we will read the registeration order, run the transformation on those files. The output of the transformation will then be merged back and then the next transformation would run. This will ensure that more than one transformation can be correctly applied to a module.

# How We Teach This

The registeration of transform is an advanced API of `ember-cli` that very few addons would use. We will be updating the guides [here](https://ember-cli.com/user-guide/#standard-anonymous-amd-asset).

# Drawbacks

The drawback of this approach is that the order of running the transformation is controlled by the addon that provides the transform rather than the addon that uses the transform. The reasoning for this is mainly for performance reasons (in order to not create a funnel per asset path) and to make sure the more than one transform can be applied correctly on an asset path.

# Alternatives

Currently the alternative is for addons to import their bower or npm dependency in `vendor` via `treeForVendor` and manually use broccoli plugins to do transformations. The alternative for apps is to create an in-repo addon to do this.

# Unresolved questions

 N/A
