- Start Date: 2018-05-22
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Deprecate Ember.{isEmpty,isBlank,isNone,isPresent}

## Summary

This RFC suggests the deprecation of the following [methods](http://api.emberjs.com/ember/release/modules/@ember%2Futils):
- `Ember.isEmpty`
- `Ember.isBlank`
- `Ember.isNone`
- `Ember.isPresent`

As well as [computed properties](http://api.emberjs.com/ember/release/modules/@ember%2Fobject) that rely on these methods:
- `notEmpty`
- `empty`
- `none`

Once deprecated, these methods will move to an external addon to allow users to still take advantage of the succinctness of these utility methods.

## Motivation

To further Ember's goal of a svelte core and moving functionality to separate packages, this RFC attempts to deprecate a few utility methods and extract them to their own addon.  A part from the aforementioned goal, in many cases, these utility methods do not add a whole lot of value over writing plain old JavaScript.  Moreover, these methods generally have a negative cognitive impact on the code in question.  In order to reduce code liability and advocate for users to write plain JavaScript, these methods could be removed from Ember core and extracted to an addon.

A simple example of this cognitive load would be checking for the presence of a variable of type String that can only have four states - `undefined`, `null`, `''` or `'MYSTRING'`.  One might check if the variable is truthy with `!!myVariable`.  However, a utility method such as `isPresent(myVariable)` can increase the cognitive load of the truthy check in question as it checks for many different types.

In addition, when checking whether a primitive value is truthy, a user can choose from `isPresent`, `isBlank`, `isNone`, or `isEmpty`.  This can can lead to some confusion for users as there are many options to do the same thing.

Lastly, `Ember.isEmpty` documented [here](https://www.emberjs.com/api/ember/release/functions/@ember%2Futils/isEmpty) will return false for an empty object.  Defining what an empty object is can be a bit tricky and `isEmpty` provided specific semantics that may not fit a users definition.  This idea of specific semantics that do not fit a users definition also applies to the other utility methods as well.

## Detailed design

`Ember.isEmpty`, `Ember.isBlank`, `Ember.isNone`, `Ember.isPresent`, `notEmpty`, `empty`, and `none` will be deprecated at first with an addon that extracts these methods for users that still find value in these utility methods.  Any instantiation of one of these functions will log a deprecation warning, notifying the user that this method is deprecated and provide a link to the supported addon.

Addons that previously relied on these utility methods would also be expected to install this package. Moreover, to ease adoption of the external addon, a codemod will become available for the community.

## How we teach this

Since using Ember in IE9, IE10, and PhantomJS is deprecated as of the 3.0 release and ES5/ES6 brings improved language semantics, we have, as a community, a guarantee of results across browsers when using plain old JavaScript.  Some of the benefits from these utility methods eminated from a need to guarantee results with ES3 browsers.  Today, we can nudge users to use JavaScript where needed and install an addon in cases where you feel it is necessary.

This would require an update to the guides to indicate that the use of any of these utility methods are now available in an addon.  Moreover, in the README of the addon, we can provide specific examples of when using JavaScript is easier than reaching for one of these utility methods.  Lastly, we can also point them to a utility library like [lodash](https://lodash.com/docs).

For many users with existing codebases, it should be as simple as installing the extracted addon.  However, users not utilizing these methods will not have these methods bundled by default.

## Drawbacks

This change may impact quite a few codebases.  Moreover, users who heavily rely on these utility methods will have to install another addon.  Some users may desire this functionality out of the box.

Given the example above, some users see a lot of value for the safety checks `isPresent(myVariable)` provides.  The type of variable could be of type String or of type Object and wiring this up to check manually is laborious.

## Alternatives

Another option would be to improve existing utility cases such as `isEmpty({}) === true`.  Moreover, we could also consider scoping existing utility methods to specific types. For example, `isEmpty` would be specific to checking objects, collections, maps or sets to reduce confusion of when these utility methods should be used.

## Unresolved questions

