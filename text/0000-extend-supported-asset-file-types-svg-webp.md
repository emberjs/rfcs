- Start Date: 2020-07-15
- Relevant Team(s): Ember CLI
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Extend the supported asset file types to include svg and webp (& possibly other)

## Summary

Support commonly used image formats svg and webp out of the box.

## Motivation

A default ember app should support commonly used image formats without requiring advanced changes to the config.

## Detailed design

Ember uses [broccoli-asset-rev](https://github.com/rickharrison/broccoli-asset-rev) to generate the appropriate urls within the ember application. The default configuration includes the following file types: `['js', 'css', 'png', 'jpg', 'gif', 'map']`. These days `svg` images are regularly used for vector graphics and `webp` is used in order to save bandwith (where supported). However, if these files are added to a new ember application, they are treated differently than the ones listed above. Fingerprinting is not enabled and related configurations like `prepend` are ignored. This can lead to anything from images not being displayed up to difficult to find issues like images being loaded from the wrong server, or broken cache invalidation.

This RFC proposes to add `svg` and `webp` to the default file types handled by broccoli-asset-rev.

## How we teach this

Not necesary. Better image support - less things to teach.

## Drawbacks

Some users might have been unaware of the possibility to configure these file types to behave correctly and might have implemented other things based on that beheaviour. There might be a possibility for these things to break when the defaults are changed. To avoid this, a deprecation warning could be implemented before changing the defaults. However, it is unclear if and how many existing users would even be affected by such a change.

## Alternatives

Add more documentation. The cli documentation currently lists this option, but it is listed under `advanced-use`. As using a svg image is pretty much a trivial thing, this should be reflected accordingly within the docs.

## Unresolved questions

Do we need some sort of deprecation warning?
