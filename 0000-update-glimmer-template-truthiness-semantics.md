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

# Update Glimmer Template Truthiness Semantics

## Summary

Provide a path for Glimmer templates to switch away from classic Handlebars truthiness to JavaScript’s truthiness. This will align our templating language with users' expectations and make teaching easier. It will also allow TypeScript features like type narrowing to work as users would expect it to with keywords like `{{and}}` or `{{or}}`.

## Motivation

Due to Handlebars supporting a different truthiness semantics than JavaScript, TypeScript can't function like users would expect it to. 
By aligning Glimmer's version of truthiness semantics with the wider javascript community, type checking will function as users expect. 

To give an example of why this is an issue we can use the `{{and}}` operator. Under the current Glimmer semantics, we can handle TypeScript in one of two ways:
1) `{{and}}` compiles to `&&` which will allow TS to lie to us about what is safe and what isn't (and will cause compile time errors) but WILL allow for type guards and narrowing to work
2) `{{and}}` compiles to `and()` which will allow truthy and falsy values to respect handlebars version of truthy and falsy, but wont allow for type guards and narrowing

## Detailed design

Modify `{{and}}`, `{{or}}`, `{{not}}`, `{{if}}` and `{{unless}}` helpers. Clarify internal workings of `{{eq}}`, `{{neq}}`, `{{lt}}`, `{{lte}}`, `{{gt}}`, `{{gte}}`

By changing support of these helpers from handlebars truthiness semantics to javascript's truthiness semantics we allow typescript to narrow properly and more closely align ourselves with the main JS community.

Our definition of falsy would then become `type Falsy = false | 0 | '' | null | undefined;` as compared to handlebars version of `Falsy = false | 0 | '' | null | undefined | [];`

### {{and}}
Takes at least two positional arguments. Raises an error if invoked with less than two arguments. It evaluates arguments left to right, returning the first one that is not truthy (by javascript's definition of truthiness) or the right-most argument if all evaluate to truthy. This is equivalent to the {{and}} helper from ember-truth-helpers. Doing this would allow `{{and}}` to compile down to `&&` and give us the correct type safety and narrowing that users would expect.

### {{or}}
Takes at least two positional arguments. Raises an error if invoked with less than two arguments. It evaluates arguments left to right, returning the first one that is truthy (by javascript's definition of truthiness) or the right-most argument if all evaluate to falsy. This is equivalent to the {{or}} helper from ember-truth-helpers. Doing this would allow `{{or}}` to compile down to `||` and give us the correct type safety and narrowing that users would expect.

### {{not}}
Unary operator. Raises an error if invoked with more than one positional argument. If the given value evaluates to a truthy value (by javscript's definition of truthiness), the false is returned. If the given value evaluates to a falsy value (by javascript's definition of truthiness) then it returns true. This is equivalent to the {{not}} helper from ember-truth-helpers. Doing this would allow `{{not}}` to compile down to `!` and give us the correct type safety and narrowing that users would expect.

### {{if}}
Same implementation as current `if`. The only difference is when doing the check for truthiness it would follow javascript's truthiness semantics.

### {{unless}}
Same implementation as current `unless`. The only difference is when doing the check for truthiness it would follow javascript's truthiness semantics.

### {{eq}}
Binary operation. Throws an error if not called with exactly two arguments. Uses triple equal version of javascript equality. Doing this allows for `{{eq}}` to compile down to `===` and give us the correct type safety and narrowing that users would expect

### {{neq}}
Binary operation. Throws an error if not called with exactly two arguments. Uses triple not equal version of javascript equality. Doing this allows for `{{neq}}` to compile down to `!==` and give us the correct type safety and narrowing that users would expect

### {{lt}}
Binary operation. Throws an error if not called with exactly two arguments. Equivalent of `<` in Javascript.

### {{lte}}
Binary operation. Throws an error if not called with exactly two arguments. Equivalent of `<=` in Javascript.

### {{gt}}
Binary operation. Throws an error if not called with exactly two arguments. Equivalent of `>` in Javascript.

### {{gte}}
Binary operation. Throws an error if not called with exactly two arguments. Equivalent of `>=` in Javascript.

### Migration Plan
1. Ship comparison helpers `and`, `or`, `not` with existing equality and truthiness semantics (they currently match `if` and `unless` truthiness semantics)
2. Ship types which match that and document why they work the way they do. Acknowledge that this wont work for TS users in the way they would expect and why.
3. Ship ability to opt into new behavior for `and`, `or`, `not`, `if` and `unless` (Following JS truthiness semantics)
4. Ship types that match this new behavior and document why you would have to opt into new behavior
5. As soon as the opt-in ability is shipped and we have had an LTS release - deprecate the old semantics and plan to remove them in the next major version.


## How we teach this

Clearly draw attention to the differences between handlebars truthiness and javascript's in the docs. Explain why we made such a choice and what benefits it gives the end user. 

## Drawbacks

We further separate ourselves from handlebars by not supporting `[]` as a falsy value. This makes the learning curve for `HTMLBars` (our version) just a little steeper when coming from handlebars

## Alternatives

One alternative involves supporting `[]` as falsy value and just trying to teach users that your types wont be correct or narrow properly.

## Unresolved questions

