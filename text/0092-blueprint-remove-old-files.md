- Start Date: 2016-12-17
- RFC PR: (leave this empty)
- Ember CLI Issue: (leave this empty)

# Summary

Give blueprint generators the ability to clean up old files.

# Motivation

We want to eliminate the noise of having old files laying after updating
ember-cli using `ember init`.

# Detailed design

We'd like an API for blueprints to delete files instead of only
create. It would be essentially syntactic sugar for removing the file yourself
in an `afterInstall` hook. It would be a returned array on the blueprint's `index.js`.

```js
// ember-cli/bluprints/blah/index.js
module.exports = {
  // ...

  get oldFilesToRemove() {
    return [
      'brocfile.js',
      'LICENSE.MD',
      'testem.json'
    ];
  }
};
```

# How We Teach This

The guides could use this addition in the blueprints section, but I envision it
being used by mostly power users.

A changelog entry should be sufficient to teach this.

# Drawbacks

The only reason to not do this is to hold out for a large blueprint reworking.
We would be locked into this API.

# Alternatives

The key name can be bikeshed. I chose `oldFilesToRemove` to be verbose and
explicit, but it can be changed.
