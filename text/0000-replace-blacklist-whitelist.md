- Start Date: 2020-06-15
- Relevant Team(s): Ember CLI, Learning
- RFC PR: [#639](https://github.com/emberjs/rfcs/pull/639)
- Tracking: (leave this empty)

# Replace terms blacklist & whitelist in Ember CLI

## Summary

Ember.js prides itself (rightly) on being an inclusive framework. To further improve on this, we should remove the terms "blacklist" and "whitelist" with more neutral replacements.

## Motivation

The terms "blacklist" and "whitelist" can be considered racially insenstive. While the origin of these terms in this context is not in itself racially motivated, the fact that in todays context it _can be considered racially insensitive_ has been discussed frequently - see for example [here](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6148600/), [here](https://bugs.chromium.org/p/chromium/issues/detail?id=981129#c16) or [here](https://www.zdnet.com/article/uk-ncsc-to-stop-using-whitelist-and-blacklist-due-to-racial-stereotyping/).

Other projects have already taken similar steps, for example [Go](https://go-review.googlesource.com/c/go/+/236857/), [Android](<https://android-review.googlesource.com/q/topic:%22soong_inclusive_language%22+(status:open%20OR%20status:merged)>), [curl](https://github.com/curl/curl/pull/5546), [PHPUnit](https://github.com/sebastianbergmann/phpunit/issues/4275), and many more.

These terms are used in Ember CLI for rather advanced functionality, which should make providing a better alternative naming and deprecating the current one rather easy.

## Detailed design

You can use `blacklist` and `whitelist` to include/exclude certain addons from your build:

https://ember-cli.com/user-guide/#whitelisting-and-blacklisting-assets

We should replace these with `exclude` or `include`:

```js
// Before
let app = new EmberApp(defaults, {
  addons: {
    blacklist: ["fastboot-app-server"],
  },
});

// After
let app = new EmberApp(defaults, {
  addons: {
    exclude: ["fastboot-app-server"],
  },
});

// Before
let app = new EmberApp(defaults, {
  addons: {
    whitelist: ["ember-cli-sass"],
  },
});

// After
let app = new EmberApp(defaults, {
  addons: {
    include: ["ember-cli-sass"],
  },
});
```

The old keys (`blacklist` and `whitelist`) should be aliased to the new ones, and show a deprecation warning. Functionally, nothing changes.

## How we teach this

We should update the Ember CLI API docs with the new terms.

## Drawbacks

This might be considered "churn" by some. However, it is a small enough change to not really impact anybody meaningfully, while ensuring that Ember.js feels inclusive to everybody.

## Alternatives

There are also other possible terms that could be used to replace `blacklist` and `whitelist`. For example:

- `allowlist` / `denylist`
- `allowlist` / `disallowlist`
- `allow` / `deny`
- `allowedlist` / `disallowedlist`
- `safelist` / `blockedlist`

## Unresolved questions

- Should we use different replacements than `exclude`/`include`?
