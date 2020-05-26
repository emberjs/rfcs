- Start Date: 2020-05-21
- Relevant Team(s): All
- RFC PR: https://github.com/emberjs/rfcs/pull/630
- Tracking: (leave this empty)

# New Browser Support Policy

## Summary

> In dropping support for IE11 at some point in the future, we will need a new
browser support policy. This is the proposed policy for that point in the future.
By properly documenting our browser support policy,
we increase our flexibility and provide explicit clarification of intended support.

## Motivation

> It should be clear which browsers are supported by Ember.js,
to eliminate any confusion and decrease the number of decisions
we're required to make about any feature (existing or new).

## Existing Policy

The existing policy is listed here for documentation purposes.

- Desktop

    1. Google Chrome, Mozilla Firefox, MS Edge, Mac Safari

        - The current and previous stable releases are supported.

    2. Internet Explorer

        - Only Internet Explorer 11 is supported.

- Mobile

    1. All

        - All mobile browsers may work but are not explicitly supported.

Any browsers not listed here may work but are not explicitly supported.

## Proposed policy

This policy drops support for a major browser, and therefore can only be
implemented with a major version bump.

- Desktop

    1. Google Chrome

        - The current and previous stable releases are supported

    2. Mozilla Firefox

        - The current and previous rapid releases and any [ESR](https://support.mozilla.org/en-US/kb/choosing-firefox-update-channel) versions currently supported by Mozilla are supported

    3. MS Edge

        - The current and previous stable releases are supported.

    4. Mac Safari

        - The versions coming with the current and previous releases of MacOS are supported, as well as versions of Mac Safari available with updates to the current release.

- Mobile

    1. Google Chrome

        - The versions coming with all versions of Android released in the past 36 months are supported, as well as the current and most stable releases available via update.

    2. iOS Safari

        - The versions coming with all versions of iOS released in the past 36 months are supported.

    3. Android

        - The versions released with all versions of Android released in the past 36 months are supported.

All browsers not listed here may work but are not explicitly supported.

## How we teach this

- A blog post explaining the policy with a link to this RFC will be posted.
- Browser support policy will be linked in repo
- Browser support policy will be added to the website
- Browser support policy will have an announcement in the Ember Times

## Drawbacks

- The proposed policy adds significant testing work to make up for
the dropping of IE11 support. This is more than made up for by clarity.
In practice our users need to support many of these browsers
anyway, and without Internet Explorer 11 compatibility requirement
it is likely that older mobile browsers will become the new limits
that prevent us from taking advantage of new features.

## Alternatives

- Keep things as they are.
- Document the existing policy but do not change it.
- Support fewer browsers, for a shorter amount of time.
- Support more browsers, for a longer amount of time.

## Unresolved questions

> Should a version of non Chromium MS Edge be supported, and if so for how long?
