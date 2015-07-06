- Start Date: 2015-07-06
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Make jQuery optional.

# Motivation

Why are we doing this? What use cases does it support? What is the expected outcome?\

I use jQuery heavily and like it! But decoupling it from Ember have some benefits: it's not always needed,
some people think payload size is important (I don't) and it may have minor performance contributions.

# Detailed design

I've tried to track down the places where jQuery is used and its implications.
There is some primary usage (around events) as well as some auxiliary usage, where it should be fairly easy
to replace it (I hope).

**Primary use** *(will require insight from people with good knowledge of browser difference)*

- [ ] For event dispatching [(code)](https://github.com/emberjs/ember.js/blob/680f997ed0958c420abdcd0b1673111aee26afe7/packages/ember-views/lib/system/event_dispatcher.js)
- [ ] *Have I missed something?*

**Auxiliary use** *(these should be easier to replace)*

- [ ] basic DOM-operations for bootstrap [(code)](https://github.com/emberjs/ember.js/blob/680f997ed0958c420abdcd0b1673111aee26afe7/packages/ember-htmlbars/lib/system/bootstrap.js)
- [ ] Firefox 11 and outerHTML[1]. Is this still needed? Firefox is on v38 now. [(code)](https://github.com/emberjs/ember.js/blob/680f997ed0958c420abdcd0b1673111aee26afe7/packages/ember-views/lib/compat/render_buffer.js#L577)
- [ ] #findElementInParentElement [(code)](https://github.com/emberjs/ember.js/blob/49d58530527c9025b4bafe4f937b0c78cfe3818e/packages/ember-views/lib/views/view.js#L1133)
- [ ] #replaceIn [(code)](https://github.com/emberjs/ember.js/blob/49d58530527c9025b4bafe4f937b0c78cfe3818e/packages/ember-views/lib/views/view.js#L1038)
- [ ] #appendTo [(code)](https://github.com/emberjs/ember.js/blob/49d58530527c9025b4bafe4f937b0c78cfe3818e/packages/ember-views/lib/views/view.js#L961)
- [ ] event listening for location changes and a DOM-lookup [(code)](https://github.com/emberjs/ember.js/blob/680f997ed0958c420abdcd0b1673111aee26afe7/packages/ember-routing/lib/location/history_location.js)
- [ ] getElementById lookup [(code)](https://github.com/emberjs/ember.js/blob/680f997ed0958c420abdcd0b1673111aee26afe7/packages/ember-views/lib/views/states/has_element.js#L25)

**Library and access** *(these should remain even if the library is optional, with a friendly message if jQuery is not present)*

- [ ] For `Ember.$`. [(code)](https://github.com/emberjs/ember.js/blob/680f997ed0958c420abdcd0b1673111aee26afe7/packages/ember-views/lib/main.js#L55)
- [ ] Core library registration [(code)](https://github.com/emberjs/ember.js/blob/680f997ed0958c420abdcd0b1673111aee26afe7/packages/ember-application/lib/system/application.js#L1141)
- [ ] The view wrapper `this.$()` [(code 1)](https://github.com/emberjs/ember.js/blob/49d58530527c9025b4bafe4f937b0c78cfe3818e/packages/ember-views/lib/views/view.js#L918) and [(code 2)](https://github.com/emberjs/ember.js/blob/680f997ed0958c420abdcd0b1673111aee26afe7/packages/ember-views/lib/views/states/has_element.js#L16)

Please add to the list if I've missed something.

Since jQuery is good at patching over browser-difference, care must be taken to understand where it's being
relied on for this.

***

I'll not be able to make all the changes myself, so I'll depend on others helping out.

Instead of tearing it all out, a good idea would probably be to remove some of the smaller jQuery usage first, with
separate PRs.

Then use a feature-flag to toggle the larger stuff, e.g. event handling with/without jQuery,
since these will need much more testing to find bugs or issues.

# Drawbacks

- It works today, why bother.
- jQuery patches over important browser differences.

# Alternatives

Wait 1-2 years, then axe jQuery.

# Unresolved questions

- How would the core team like to handle the 'make it optional' switch? May affect the build process,
library requirements etc.
- The FF11 requirement, what is the minimum supported version of FF? Is it fine to drop? (FF is on ~38 now)

# References

[1] https://developer.mozilla.org/en-US/docs/Web/API/Element/outerHTML
