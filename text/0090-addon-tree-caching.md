# Ember CLI Addon Deduplication

- Start Date: 2016-12-11
- RFC PR: https://github.com/ember-cli/rfcs/pull/90
- Ember CLI Issue: (leave this empty)
 
# Summary

In Ember CLI today, all addons at each level are built through the standard `treeFor` / `treeFor*` hooks. These hooks are responsible for preprocessing the JavaScript included by the tree returned from that specific hook (e.g., `treeForAddon` preprocesses the JS for the addon tree). This RFC proposes a mechanism that would allow these returned trees to be cached by default (when no build time customization is done) and expose proper hooks for addon authors to control the degree to which we dedupe these trees.

# Motivation

Today, given the dependency graph:

``` 
ember-basic-dropdown:
  ember-wormhole@0.4.1

ember-modal-dialog:
  ember-wormhole@0.4.1

ember-paper:
  ember-wormhole@0.4.1
```
We would actually build `ember-wormhole`'s `addon` tree 3 different times, even though [as you can see](https://github.com/yapplabs/ember-wormhole/blob/0.4.1/index.js) there is absolutely no build time customization being done. After all of these `ember-wormhole` tree instances are built, we merge them such that the last tree wins (thus making all of the work to preprocess these trees completely moot). If you extrapolate this out to larger applications or ones using multiple engines (lazy or not) it is fairly common to see these sorts of dependencies shared upwards of 4 to 5 times. This can lead to significant build performance degradation.

# Detailed design

- Add a `Addon.prototype.cacheKeyForTree` method to [lib/models/addon.js](https://github.com/ember-cli/ember-cli/commits/master/lib/models/addon.js) that is invoked prior to calling `treeFor` for the same tree name. The `Addon.prototype.cacheKeyForTree` method is expected to return a cache key allowing multiple builds of the same tree to simply return the original tree (preventing duplicate work). If `Addon.prototype.cacheKeyForTree` returns `null` / `undefined` the tree in question will opt out of this caching system.
- ember-cli's custom [`mergeTrees` implementation](https://github.com/ember-cli/ember-cli/blob/4ec7b5951e8a9dd292029faf20d1858abf7bdfa0/lib/broccoli/merge-trees.js) (which is already aware of other tree reduction techniques) will be updated so that calling `mergeTrees([treeA, treeA]);` simply returns `treeA`, and `mergeTrees([treeA, treeB, treeA])` removes the duplicated `treeA` in the input nodes.
 
The proposed declaration for `Addon.prototype.cacheKeyForTree` in Typescript syntax is:

``` ts
function cacheKeyForTree(treeType: string): string;
```
The default implementation for `Addon.prototype.cacheKeyForTree` will:

- Utilize a shared NPM package (e.g. `calculate-cache-key-for-tree`) that will generate a cache key that incorporates at least the following pieces of information:
    - `this.name` - The addon's name (generally from `package.json`).
    - `this.pkg` - This builds a checksum accounting for the addon's `package.json`.
    - `treeType` - The specific tree in question (e.g. `addon`, `vendor`, `addonTestSupport`, `templates`, etc).
 
- Resort to disabling all addon tree caching in the following scenarios
    - The addon implements a custom `treeFor`
    - The addon implements a custom `treeFor*` method (where `*` represents the tree type)
 
 
Addons that implement custom `treeFor` or `treeFor*` methods can still opt-in to caching in scenarios that they can confirm are safe. To do this, they would implement a custom `cacheKeyForTree` method and return a cache key as appropriate for their caching needs.

# How We Teach This

This is something that we do not expect 99% of ember-cli users to have to learn and understand, however it is still important for it to be possible to determine what is going on and how to work within the system when building addons.

The following should help us teach this to the correct audience (roughly "addon power users"):

- Document the shared NPM package (referred to above as `calculate-cache-key-for-tree`). This will help authors of addons that need to implement `treeFor*` hooks understand how they can properly implement `Addon.prototype.cacheKeyForTree`.
- Write API docs for the newly added `Addon.prototype.cacheKeyForTree` method.
 
# Drawbacks

- Cache invalidation is difficult to get right, and it is possible to accidentally troll our users. This can be mitigated by thorough review of the implementation and this RFC.
 
# Alternatives

# Unresolved questions

- Confirm if including the same tree multiple times will only trigger a single build of that tree (this should be a Broccoli feature). We have confirmed that code exists in broccoli-builder ([see here](https://github.com/ember-cli/broccoli-builder/blob/0-18-x/lib/builder.js#L89-L97)), but still need to actually confirm `.build` / `.read` / `.rebuild` are not called twice within the same build.
 
