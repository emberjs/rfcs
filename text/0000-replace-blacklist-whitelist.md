- Start Date: 2020-06-15
- Relevant Team(s): Ember CLI, Learning
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Replace terms blacklist & whitelist in Ember CLI

## Summary

Ember.js prides itself (rightly) on being an inclusive framework. To further improve on this, we should remove the terms "blacklist" and "whitelist" with more neutral replacements.

## Motivation

The terms "blacklist" and "whitelist" are considered racially insenstive. These terms are used in Ember CLI for rather advanced functionality, which should make providing a better alternative naming and deprecating the current one rather easy.

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

Leave it as it is.

## Unresolved questions

- Should we use different replacements than `exclude`/`include`?
