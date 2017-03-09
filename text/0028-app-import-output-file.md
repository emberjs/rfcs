- Start Date: 2015-11-02
- RFC PR: [#28](https://github.com/ember-cli/rfcs/pull/28)

# Summary

Allow `app.import` to specify outputFIle of a given import.
The default `app.import` would be considered to have an outputFile of
`assets/vendor.js`

# Motivation

It is common for individuals to want control over the outputFile for a given
dependency. For example, one may want to load some asm.js code independently
rather then via the single vendor.js blob.

It is also common for developers to want to group various dependencies
together, and then lazy-load them in the routes they are required.

Although not as automatic as we would like, it does provide a rather elegant
escape valve. Further work will likely continue to explore automation.

# Detailed design

`outputFile:` option, specifies the target file for the given import. If
multiple imports share an outputFile, they will be concatenated (regardless of
type, css/images/videos/js/txt) in the order they where imported.

`outputFile:` will default to `assets/vendor.js`

## Examples


#### variation 0: the default

```js
app.import('vendor/vim.js', { outputFile: 'assets/vendor.js'});
```

#### variation 1: 1 file -> 1 outputFile

```js
app.import('vendor/vim.js', { outputFile: 'assets/vim.js'});
```

* `vendor/vim.js` becomes `assets/vim.js`
* in prod it is:
  * uglified (unless using the uglify options it is excluded)
  * fingerprinted (unless it is excluded via the asset-rev options)

#### variation 2, multiple files to same outputFile

```js
app.import('vendor/dependency-1.js', { outputFile: 'assets/alternate-vendor.js'});
app.import('vendor/dependency-2.js', { outputFile: 'assets/alternate-vendor.js'});
```

* in-order of the corresponding `app.import` invocation, using sourceMap
  concat, the files are combined into `assets/alternate-vendor.js`
  * `vendor/dependency-1.js` + `vendor/dependency-2.js` >> `assets/alternative-vendor.js`

#### variation n, multiple files to same outputFile

```js
app.import('vendor/dependency-1.js', { outputFile: 'assets/alternate-vendor.js'});
app.import('vendor/dependency-2.js', { outputFile: 'assets/alternate-vendor.js'});
app.import('vendor/dependency-n.js', { outputFile: 'assets/alternate-vendor.js'});
```

* resulting concat is:
  * `vendor/dependency-1.js` + `vendor/dependency-2.js` ... `vendor/dependency-n.js`>> `assets/alternative-vendor.js`

# Drawbacks

* potential overlap with @chadhietala's packager/linker work
* does not offer additional build-pipeline hooks for these files

# Alternatives

Alternatives exist, such as adding support to the linker/packager
effort, or instructing developers to drop down and use
broccoli-funnel/source-map-concat.

The linker/packager effort is still a ways off, and could be thought of as
complementary.

Dropping down to broccoli is a solution available today, but for this problem,
it feels like a slightly too low level of abstraction.

# Unresolved questions

* how does this relate to `type` in `app.import(..., { type: ... })` ?
* should additional build-steps be allowed for specific output files? (I
  suspect maybe, but a future RFC can likely explore)
