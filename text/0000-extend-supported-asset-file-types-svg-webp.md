- Start Date: 2020-07-15
- Relevant Team(s): Ember CLI
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Extend the supported asset file types to include svg and webp

## Summary

Support commonly used image formats svg and webp out of the box.

## Motivation

A default Ember app should support commonly used image formats without requiring advanced changes to the config.

## Detailed design

Ember uses [broccoli-asset-rev](https://github.com/rickharrison/broccoli-asset-rev) to generate the appropriate urls within the Ember application. Ember's default value for `extensions` is currently the same as broccoli-asset-rev's:

```javascript
// Current default extensions
['js', 'css', 'png', 'jpg', 'gif', 'map']
```

These days, `svg` images are regularly used for vector graphics and `webp` is used in order to save bandwidth (where supported). If `svg` and `webp` files are added to a new Ember application, they are treated differently than files with extensions listed above. Fingerprinting is not enabled and related configurations like `prepend` are ignored.

This can lead to:

- Images not being displayed
- Images being loaded from the wrong server
- Broken cache invalidation

This RFC proposes to add `svg` and `webp` to the default file types handled by broccoli-asset-rev.

## How we teach this

Not necesary. Better image support - less things to teach.

## Drawbacks

Some users might have been unaware of the possibility to configure file extensions and might have implemented other things based on that behavior. There might be a possibility for these things to break when the defaults are changed.

To avoid this, a deprecation warning could be implemented before changing the defaults and/or ember-cli-update could add the current formats to the config when upgrading, so behaviour will not change within an ember cli upgrade. However, it is unclear if and how many existing users would even be affected by such a change. It seems somewhat unlikely that this will actually be an issue, but it seems reasonable to think about any possible side effects.

## Alternatives

Add more documentation. The [CLI documentation](https://cli.emberjs.com/release/advanced-use/asset-compilation/#fingerprintingandcdnurls) currently lists this option, but it is listed under `advanced-use`. As using a svg image is pretty much a trivial thing, this should be reflected accordingly within the docs.

## Unresolved questions

Do we need some sort of deprecation warning?
Are there other common file formats within web apps these days, that should be added as well?
