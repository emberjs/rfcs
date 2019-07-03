- Start Date: 2017-10-03
- RFC PR: https://github.com/emberjs/ember.js/pull/15780
- Ember Issue: (leave this empty)

# Summary

This RFC aims to deprecate usage of 
[`link-to` component](https://emberjs.com/api/ember/2.16/classes/LinkComponent)'s `disabledWhen` 
and use `disabled` property instead. 

# Motivation

In `link-to` component `disabledWhen` property currently behaves exactly like 
`disabled` property in other built-in components (like `input`), but instead `disabled` property is reserved for internal usage in `link-to`. 

Therefore in order to bring `link-to` into consistency with other built-in components I suggest to deprecate `disabledWhen` property and use `disabled` property instead. Which also makes things intuitive and straightforward both for new and old users.

# Detailed design
Follow the usual process for other deprecations

# How We Teach This
It is more intuitive to use `disabled` property because nativly supported by other html elements and therefore new user's more likely familiar with it already. Deprecation will simply suggest to use `disabled` instead of `disabledWhen`.

# Drawbacks
Implementation of deprication can be done with none breaking way so don't see any drawbacks to it.

# Alternatives
None

# Unresolved questions
None

