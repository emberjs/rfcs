- Start Date: 2015-11-11
- RFC PR: #29
- ember-cli Issue: (leave this empty)

# Summary

It should be possible to white- and/or blacklist addons in `EmberApp`.

# Motivation

If there are two (or more) `EmberApp`s, it's very likely that not all applications need all addons.
E.g. if there is a main page application and one application for embeddable widgets, the main page might need all sorts of addons like `ember-modal-dialog` which adds completely useless bytes to widgets javascript and css files for the widgets. Other addons may add useless initializers or other things that have runtime performance penalties for no benefit.

# Detailed design

When EmberApp ctor gets passed a blacklist like this

```javascript
  EmberApp({
    addonBlacklist: ['ember-modal-dialog']
  });
```

it won't add addons whose `name` matches `ember-modal-dialog` to the list of addons for this app, just as if the addon's `isEnabled()` hook returned `false`.

C.f. https://github.com/ember-cli/ember-cli/blob/master/lib/broccoli/ember-app.js#L344, this is also were I would add this check.

Whitelist could work analogously.

# Drawbacks

- It adds a bit of API surface while you (possibly) don't care for the multiple app use case of ember-cli.

- It kinda makes the `name` of an addon public api, so people might change it, not noticing they are breaking people's build (and people might not notice it either). Some addons have names like `Ember CLI ic-ajax` which seems awkward to use as an identifier.

# Alternatives

- Add a unified way to enable/disable an addon via normal config
    - if that is added one day, the white/blacklist could still be an abstraction for that

# Unresolved questions

None.
