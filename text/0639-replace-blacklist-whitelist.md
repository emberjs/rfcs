---
Start Date: 2020-06-15
Relevant Team(s): Ember CLI, Learning
RFC PR: #639
Tracking: (leave this empty)

---

# Replace terms blacklist & whitelist in Ember CLI

## Summary

Ember.js prides itself (rightly) on being an inclusive framework. To further improve on this, we should remove the terms "blacklist" and "whitelist" with more neutral replacements.

## Motivation

The terms "blacklist" and "whitelist" can be considered racially insenstive. While the origin of these terms in this context is not in itself racially motivated, the fact that in todays context it _can be considered racially insensitive_ has been discussed frequently - see for example [here](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6148600/), [here](https://bugs.chromium.org/p/chromium/issues/detail?id=981129#c16) or [here](https://www.zdnet.com/article/uk-ncsc-to-stop-using-whitelist-and-blacklist-due-to-racial-stereotyping/).

Other projects have already taken similar steps, for example [Go](https://go-review.googlesource.com/c/go/+/236857/), [Android](<https://android-review.googlesource.com/q/topic:%22soong_inclusive_language%22+(status:open%20OR%20status:merged)>), [curl](https://github.com/curl/curl/pull/5546) or [PHPUnit](https://github.com/sebastianbergmann/phpunit/issues/4275).

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

The old keys (`blacklist` and `whitelist`) should be aliased to the new ones. Functionally, they behave the same.

If both keys are used at the same time (so either `blacklist` AND `exclude` or `whitelist` AND `include`), an error is thrown.

The old keys should be removed at some point - there should be a dedicated deprecation RFC for this (at a later point), ideally in time for them to be removed in the next major release.

## How we teach this

We should update the Ember CLI API docs with the new terms.

## Drawbacks

This might be considered "churn" by some. However, it is a small enough change to not really impact anybody meaningfully, while ensuring that Ember.js feels inclusive to everybody.

## Alternatives

There are also other possible terms that could be used to replace `blacklist` and `whitelist`. For example:

- `allowlist` / `denylist`
- `allowlist` / `disallowlist`
- `allow` / `deny`
- `allowed` / `disallowed`
- `allowedlist` / `disallowedlist`
- `safelist` / `blockedlist`

For reference, this is how often these terms are used accross the Ember ecosystem according to Ember Observer - all searches done inside of index.js files for easier comparison:

- blacklist: 16 addons
- whitelist: 20 addons
- allowlist, allowedlist, denylist, deniedlist, disallowlist, disallowedlist, blockedlist: 0 addons
- blocklist: 2 addons
- safelist: 1 addon
- allow: 4 addons
- deny: 1 addon
- allowed: 18 addons
- disallowed: 1 addon
- include: 207 addons
- exclude: 101 addons

## Unresolved questions

- Should we use different replacements than `exclude`/`include`?
