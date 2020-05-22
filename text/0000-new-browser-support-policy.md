- Start Date: 2020-05-21
- Relevant Team(s): All
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# New Browser Support Policy

## Summary

> In dropping support for IE11 at some point in the future, we will need a new
browswer support policy. This is the proposed policy.

## Motivation

> It should be clear which browsers are supported by Ember.js

## Detailed policy

- Desktop

    1. Google Chrome

        - The current and previous stable releases are supported

    2. Mozilla Firefox

        - The current and previous rapid releases and the latest [ESR](https://support.mozilla.org/en-US/kb/choosing-firefox-update-channel) are supported

    3. MS Edge

        - The current and previous stable releases are supported.

    4. Mac Safari

        - The versions coming with the current and previous releases of MacOS are supported, as well as versions of Mac Safari available with updates to the current release.

- Mobile

    1. Google Chrome

        - The versions coming with all versions of Android released in the past 18 months are supported, as well as the current and most stable releases available via update.

    2. iOS Safari

        - The versions coming with all versions of iOS released in the past 36 months are supported.

    3. Android

        - The versions released with all versions of Android released in the past 18 months are supported.

All browsers not listed here may work but are not explicitly supported.

## How we teach this

> A blog post explaining the policy with a link to this RFC will be posted.

## Drawbacks

- See discussions on this RFC

## Alternatives

> See discussions on this RFC

## Unresolved questions

> Should a version of non Chromium MS Edge be supported, and if so for how long?
