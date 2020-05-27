- Start Date: 2020-05-15
- Relevant Team(s): All teams
- RFC PR: https://github.com/emberjs/rfcs/pull/627
- Tracking: (leave this empty)

# Ember 4.0

## Summary

> Ember.js, Ember-Data, and Ember-CLI will release a new major version, 4.0,
at least six months from today, possibly as soon as December 2021, but this date
may slip for a variety of reasons.
The last version in the 3.x cycle of each of these projects will be an LTS.
In order to prevent a rush of deprecations submitted to Ember prior to the major
version bump, any new proposed deprecations must target Ember 5.0.

## Motivation

> We will release a new major version of Ember in order to drop support for IE11.
There are several major reasons for doing this.

1. Supporting IE11 prevents us from using the latest web technologies in Ember,
such as native proxies, that cannot be polyfilled.

1. Supporting IE11 requires us to ship code to the client that in order to polyfill
features that IE11 does not support.

1. Supporting IE11 adds a significant maintenance overhead for the Ember project.

1. If separate code is not shipped, IE11 polyfills can cause significant performance issues.
They do not optimize well in V8 and other JavaScript engines.

1. Many addons do not support IE11 but do not specifically document this. This can lead
to confusion. Without adequate testing, many applications fail to support IE11 in practice.

1. Many community tools, such as Ember Inspector and Ember Twiddle, do not work in IE11.

> We are now able to drop support for IE11 since major enterprises such as LinkedIn
are doing so. However, we are waiting at least six months in order for companies to notify
clients and make appropriate plans. Developers that still need to support IE11 may
stay on Ember 3.x indefinitely, and it will be a supported LTS until Ember 4.4 becomes LTS,
approximately mid May 2021.

## Detailed design

> In a separate RFC, a new browser support policy will be introduced for the 4.x cycle
and possibly beyond.

> No new features will be introduced in Ember 4.0. Instead,
all deprecated functionality targeting Ember 4.0 may be removed following its release.

> There will be special handling for imports from `@ember/polyfills`. Specifically,
these will not emit deprecation warnings by default in the Ember 3.x cycle so that both
apps and addons can continue to work with IE11. A way to turn on deprecation warnings
in some versions of Ember 3.x may be provided in a future RFC.
In Ember 4.0, these functions will be replaced by stubs which emit a deprecation warning
before delegating to native methods.
These deprecated stubs will continue to exist until Ember 4.4 to give time for apps and addons
to remove them replace them with native methods.
They may be removed beginning in Ember 4.5.
Please note that these imports shall not actually work in IE11 in Ember 4.x; they exist
primarily to ease code migration.

## How we teach this

A blog post will be released announcing this as soon as this RFC is accepted, detailing
all relevent information, similar to the one announcing
[Ember 3.0](https://blog.emberjs.com/2017/10/03/the-road-to-ember-3-0.html).

## Drawbacks

> We may push some companies which must continue to support IE11 into looking into
alternative frameworks.

## Alternatives

> We can continue to support IE11 for a longer period of time.

## Unresolved questions

> How can we allow some applications and/or addons to opt into receiving deprecation
warnings for `@ember/polyfills` in the remainder of the 3.x cycle?
