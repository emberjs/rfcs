---
Start Date: 2016-04-18
RFC PR: #139
Ember Issue: emberjs/ember.js#13666

---

# Summary

Introduce `Ember.String.isHtmlSafe()` to provide a reliable way to determine if an object is an "html safe string", i.e. was it created with `Ember.String.htmlSafe()`.


# Motivation

Using `new Ember.Handlebars.SafeString()` is slated for deprecation. Many people are currently using the following snippet as
a mechanism of type checking: `value instanceof Ember.Handlebars.SafeString`. Providing `isHtmlSafe` offers a
cleaner method of detection. Beyond that, the aforementioned test is a bit leaky. It requires the developer to understand
`htmlSafe` returns a `Ember.Handlerbars.SafeString` instance and thus limits Ember's ability to change
`htmlSafe` without further breaking it's API.

Based on our app at Batterii and some research on Github, I see two valid use cases for introducing this API.

First, and most commonly, is to make it possible to test addon helpers that are expected to return a safe string. I believe this test on ember-i18n says it all: ["returns HTML-safe string"](https://github.com/jamesarosen/ember-i18n/blob/master/tests/unit/utils/i18n/default-compiler-test.js#L56-L59).

The second use case is to do type checking. In our app, we have an `isString` utility that is effectively:

```javascript
import Ember from 'ember';

export default function(value) {
  return typeof value === 'string' || value instanceof Ember.Handlebars.SafeString;
}
```

Newer versions of ember-i18n, doing `this.get('i18n').t('someTranslatedValue')` will return a safe string. Thus our `isString` utility has to consider that.


# Detailed design

`isHtmlSafe` will be added to the `Ember.String` module. The implementation will basically be:

```javascript
function isHtmlSafe(str) {
  return str && typeof str.toHTML === 'function';
}
```

It will be used as follows:

```javascript
if (Ember.String.isHtmlSafe(str)) {
  str = str.toString();
}
```


# Transition Path

As part of landing `isHtmlSafe` we will simultaneously re-deprecate `Ember.Handlebars.SafeString`. This deprecation will
take care to ensure that `str instanceof Ember.Handlebars.SafeString` still passes so that we can continue to
maintain backwards compatibility.

Additionally, a polyfill will be implemented to help provide forward compatibility for addon maintainers and others
looking to get a head while still on older versions of Ember. Similar to [ember-getowner-polyfill](https://github.com/rwjblue/ember-getowner-polyfill).


# How We Teach This

I think we'll continue to refer to these strings as "html safe strings". This RFC does not
introduce any new concepts, rather it builds on an existing concept.

I don't believe this feature will require guide discussion. I think API Docs will suffice.

The concept of type checking is a pretty common programming idiom. It should be relatively self
explanatory.


# Drawbacks

The only drawback I see is that it expands the surface of the API and it takes a step
towards prompting "html safe string" as a thing.


# Alternatives

An alternative would be to expose `Ember.Handlerbars.SafeString` publicly once again. Users
could revert back to using `instanceof` as their type checking mechanism.


# Unresolved questions

There are no unresolved questions at this time.
