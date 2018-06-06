- Start Date: 2017-10-03
- RFC PR: https://github.com/emberjs/ember.js/pull/15780
- Ember Issue: (leave this empty)

# Summary

This RFC aims to deprecate usage of 
[`link-to` component](https://emberjs.com/api/ember/2.16/classes/LinkComponent)'s `disabledWhen` property and use `disabled` property instead. 

# Motivation

In `link-to` component `disabledWhen` property currently behaves exactly like 
`disabled` property in other built-in components (like `input`), and while `disabled` property is reserved for internal usage. 

Therefore in order to bring `link-to` into consistency with other built-in components I suggest to deprecate `disabledWhen` property and use `disabled` property instead. Which also makes things intuitive and straightforward both for new and old users.

# Detailed design

Follow the usual process for other deprecations.

# How We Teach This

Deprecation will suggest usage of `disabled` and removal of `disabledWhen`

# Drawbacks
None

# Alternatives
None

# Unresolved questions

