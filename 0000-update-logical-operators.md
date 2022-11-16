---
stage: accepted
start-date: 11/16/2022
release-date: 
release-versions:
teams: # delete teams that aren't relevant
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: # update this to the PR that you propose your RFC in
project-link:
---

<!--- 
Directions for above: 

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
-->

# <RFC title>

## Summary

As part of RFC 0562, logical operators `and`, `or` and `not` were added to Ember.js. They need to be updated to follow javascript's version of truthy and falsy. 

## Motivation

Due to the original RFC supporting a different version of truthy/falsy than javascript(i.e. handlebars version), we would need to make a no win decision with two options. 

1) `{{and}}` compiles to `&&` which will allow TS to lie to us about what is safe and what isn't (and will cause compile time errors) but WILL allow for type guards and narrowing to work
2) `{{and}}` compiles to `and()` which will allow truthy and falsy values to respect handlebars version of truthy and falsy, but wont allow for the types to be correct

The same issues follow for the `{{or}}` and `{{not}}` helpers. 

## Detailed design

Modify {{and}}, {{or}} and {{not}} helpers.

By changing support of these helpers from handlebars version of truthy/falsy to javascript's version we allow typescript to narrow properly and more closely align ourselves with the main JS community.

Our definition of falsy would then become `type Falsy = false | 0 | '' | null | undefined;` (as compared to `Falsy = false | 0 | '' | null | undefined | [];` from RFC 0562)

### {{and}}
Takes at least two positional arguments. Raises an error if invoked with less than two arguments. It evaluates arguments left to right, returning the first one that is not truthy (by javascript's definition of truthiness) or the right-most arguments if all evaluate to truthy. This is equivalent to the {{and}} helper from ember-truth-helpers. Doing this would allow `{{and}}` to compile down to `&&` and give us the correct type safety and narrowing that users would expect.

### {{or}}
Takes at least two positional arguments. Raises an error if invoked with less than two arguments. It evaluates arguments left to right, returning the first one that is truthy (by javascript's definition of truthiness) or the right-most argument if all evaluate to falsy. This is equivalent to the {{or}} helper from ember-truth-helpers. Doing this would allow `{{or}}` to compile down to `||` and give us the correct type safety and narrowing that users would expect.

### {{not}}
Unary operator. Raises an error if invoked with more than one positional argument. If the given value evaluates to a truthy value (by javscript's definition of truthiness), the false is returned. If the given value evaluates to a falsy value (by javascript's definition of truthiness) then it returns true. This is equivalent to the {{not}} helper from ember-truth-helpers. Doing this would allow `{{not}}` to compile down to `!` and give us the correct type safety and narrowing that users would expect.


## How we teach this

While the introduction of these helpers doesn't introduce new concepts, as helpers like these could be written and in fact were written for a long time, it might affect slightly how we frame some concepts in the guides.

Previously users were encouraged to put computed properties in the javascript file of the components, even for the most simple tasks like negating a boolean condition using computed.not or adding them with computed.and.

With the addition of these helpers users don't have to resort to computed properties for simple operations, which sometimes forced users to create javascript files for what could have been template-only components.

In addition to documenting the new helpers in the API docs, the Guides should be updated to favour the usage of helpers over computed properties where it makes more sense, adding illustrative examples and stressing out where the definition of truthiness of handlebars differs from the one of Javascript.

## Drawbacks

We further separate ourselves from handlebars by not supporting `[]` as a falsy value. This makes the learning curve for `HTMLBars` (our version) just a little steeper when coming from handlebars

## Alternatives

One alternative involves supporting `[]` as falsy value and just trying to teach users that your types wont be correct or narrow properly.

## Unresolved questions

