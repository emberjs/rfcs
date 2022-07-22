---
Stage: 
Start Date: 
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Learning team, CLU
RFC PR: 
---

<!--- 
Directions for above: 

Stage: Leave as is
Start Date: Fill in with today's date, YYYY-MM-DD
Release Date: Leave as is
Release Versions: Leave as is
Relevant Team(s): Fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies
RFC PR: Fill this in with the URL for the Proposal RFC PR
-->

# <RFC title>
Standardize the use of yarn and npm scripts in the Ember experience

## Summary

This change encourages developers to use `npm` or `yarn` 
for certain commands when working with Ember
applications, rather than using global Ember CLI commands like `ember serve`.
This aligns Ember with norms in the JavaScript community, and
helps in reducing the confusion around Ember-specific commands.

## Motivation

In many JavaScript projects, the following commands are very common:

```
npm start
npm test
```

These scripts are defined in the `package.json` of Ember apps, however,
Ember's documentation tells developers to run these commands instead:

```
ember serve
ember test
```

Notably, `ember test` and `npm test` give different results.

When we run `ember test`, it sets the environment to test and performs Ember.onerror validation by default.
Whereas, In the case of `npm test`, there is an abstraction of the underlying commands that allows the user to run extra checks such as lint tests across the files  and finally performs `ember test`.
This is useful in carrying on a sequence of instructions and using them without having to worry about how they function behind the hood.
As a result, a lot of developer time is  saved and it also reduces the human error that might have been made if the abstract tooling was not used.

`ember test`
![Ember test](/images/ember_test.png)

`npm test` 
![Npm test](/images/npm_test.png)

If documentation encouraged using `yarn` or `npm`, this would allow developers to customize the scripts
themselves while also having a standard command that everyone can run in any project 
and get an expected output, regardless of what's going on under the hood.

Consider cases where the author of an addon sets up yarn test to run with ember-exam.
Un such cases, one shouldn't be manually changing the default documentation for a script that already existed.

In another case of using the `package.json` script for start/test, it allows teams to abstract details about their specific dev environment,
which makes developers' jobs easier. They can use "yarn start" or 
"pnpm start" without actually having to know which command starts the server.
Moreover, using tooling abstraction provided by npm/yarn helps in staying consistent with industry standards rather than having to use bespoke
tools.

## Detailed design

This will involve two steps

1. We should decide whether we want a Standardize the use of yarn/npm scripts.

2. Then we make the changes in readmes, contributing guides, CLI output, and in learning docs.

## How we teach this

The following resources would need to change:

* Ember-cli guides
* Ember guides 
* Ember API documentation 
* The Super Rentals tutorial
* Readme.md and contributing.md of repos
* Blueprints in ember-cli

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.
https://github.com/ember-cli/ember-cli/issues/8969#issuecomment-1167894022

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

* No change

## Unresolved questions

*  How does the difference between yarn 1, 2, and 3 affect us if we made this change?
*  Should we show yarn and npm for every example?