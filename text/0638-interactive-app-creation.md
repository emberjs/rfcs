---
Start Date: 2020-06-12
Relevant Team(s): ember-cli, framework, learning
RFC PR: https://github.com/emberjs/rfcs/pull/638
Tracking: (leave this empty)

---

# Interactive New Ember App Creation

## Summary

As part of the effort to make new Ember apps more conformant for digital accessibility requirements at a global scale, this RFC proposes an interactive workflow for new Ember apps. This will also have the benefit of assisting new users who prefer an interactive model of new app creation. 

## Motivation

This RFC is the result of [analysis and discussion in Ember's accessibility strike team](https://github.com/ember-a11y/core-notes/blob/ember-a11y/ember-a11y/2020-05/may-20.md). The overall roadmap for addressing this issue is with a series of RFCs that intend to (independently) offer: 
* a partial resolution to ensuring new Ember apps achieve [WCAG Success Criteria 3.1.1 (language of the page)](https://www.w3.org/WAI/WCAG21/Understanding/language-of-page.html)
* enhancements to Ember CLI that empower developers to make their applications more accessible-by-default
* global improvements to new Ember applications (i.e., improvements gained by achieving WCAG SC-3.1.1 that aren't specifically related to digital accessibility)

The underlying motivation for this RFC is the desire to improve the accessibility of new Ember apps as [previously documented in Issue 595/Section 4](https://github.com/emberjs/rfcs/issues/595). Specifically, it aims to prevent new Ember apps from failing legal requirements for accessibility conformance according to [WCAG 2.1](https://w3c.github.io/wcag/21/guidelines/), the standard used by most countries around the world to inform digital accessibility requirements. In that context, this RFC seeks to help new Ember applications achieve [WCAG Success Criteria 3.1.1: language of page](https://www.w3.org/TR/WCAG21/#language-of-page) by providing users of `ember-cli` to specify the base human language of their new application within an interactive setup workflow.

That being said, the overall developer experience improvement is the primary motivation. Interactive app creation helps new developers understand the important decisions they must make, and safely explore the options that they have (as defined in the question options). This provides a learning environment that will help build up the confidence of the new Ember developer by providing them with an interactive environment and helpful feedback during troubleshooting and error handling, thus flattening the learning curve. This will also help more experienced developers move faster; the strong defaults that Ember is known for will make it even easier than before to set up a new repo with the confidence that the important things are not being forgotten. 

## Detailed design

The first step of the solution was presented in [RFC 635](https://github.com/emberjs/rfcs/pull/635)-- the addition of the `--lang` flag as a way to add the language attribute to the `<html>` tag in the `index.html` file of new Ember apps. As such, successful implementation of [RFC 635](https://github.com/emberjs/rfcs/pull/635) (or similar) is a necessary pre-requisite for this RFC, which in turn proposes an interactive workflow that can be initiated in the CLI when creating a new Ember app. With this in mind, this RFC will endeavor to not repeat the information in that RFC, but rather build on it, and reference when appropriate. The intention of [RFC 635](https://github.com/emberjs/rfcs/pull/635) was to provide the appropriate underlying mechanism for setting the page language to align new Ember apps with WCAG Success Criteria 3.1.1 (language of the page); the interactive workflow proposed in **this** RFC intends to harness that mechanism.

### Prior Art
It would be remiss to introduce this RFC without mentioning the prior art that we analyzed for wizard-like app creation. In the Ember community, [@gossi](https://gist.github.com/gossi) first introduced this idea with [ember-cli-create](https://github.com/gossi/ember-cli-create). In the wider web community, the Vue community offers this through `vue create` ([see docs](https://cli.vuejs.org/guide/creating-a-project.html#vue-create)).

### Intro 
When considering what questions to ask, we considered a few things: 
1. What questions do we have to ask ourselves every time we create a new Ember app?
1. How many questions do we think developers will tolerate?
1. Should the questions be different if it is an app vs addon?

We will also endeavor to provide strong defaults for these questions, based on the results from the community surveys we have been conducting over the past few years and the defaults as stated by Ember itself (either explicitly or implicitly). **If these defaults change over time as the result of community RFCs, the default values or other responses for the interactive questions should be updated as part of those RFCs.**

So, how then do we decide which questions to include and which not to include? It became necessary to then develop a rubric by which the proposed list of questions could be measured. 

### Question Rubric
We intend for this section to be normative; that is, we intend for it to define how questions are evaluated for inclusion both now and in the future. 

Evaluating criteria: 
1. Is it a choice that is hard (or time-consuming) to change one's mind about later? *For example, it's easy to remove the `ember-welcome-page` addon, so we did not include it. But it's time consuming to change from `yarn` to `npm` (or vice-versa) so we made sure to include that.* 
1. Is it difficult to discover the option?
1. What is the maintenance cost? *For example, a nice question to add could be something like, Do you want to use a CSS preprocessor? But, because Ember does not deal with SASS, LESS... by default, the interactive workflow will have to install an addon (like `ember-cli-less`) which would imply that the CLI team will have to ensure it never fails.* 
1. Does the question affect another one? *For example, the LTS question can affect the CI file generated by the CI question (setup tests on CI with `ember-try`). Questions that modify the result of other questions too much should be avoided to prevent high maintenance.*

### Triggering the Workflow

The following commands, and reasonable combinations thereof, should work to enter into the interactive workflow: 

1. `ember new`
1. `ember new --interactive`
1. `ember new -i`
1. `ember new my-app -i`

### App/Addon 

Starting a new Ember app or addon as we do now (at the time of this writing) will not change. If a user types `ember new my-app` they will still get a new Ember app generated in the usual ways. However, if the user types `ember new` they will enter into an interactive workflow that will ask them a few key questions about their application. 

If a user were to type in existing flags that are also questions in the interactive workflow (i.e., `ember new my-app --lang en-US --interactive`), it will skip the question entirely. 

When a user types `ember new` into their command line, they will be asked these questions (defaults indicated by a `>`):

- `Is this an app or an addon? (use arrow keys)`
  - `> app`
  - `addon`
- `Please provide the name of your app/addon:`
- `Please provide the spoken/content language of your app/addon: (use arrow keys)`
  - `> en-US`
  - `my computer's default language`
  - `manually define a different language`
- `Pick the package manager to use when installing dependencies:  (use arrow keys)`
  - `> NPM`
  - `Yarn`
  - `ignore/skip`
- `What CI system do you want to use? (use arrow keys)`
  - `> travis-ci`
  - `ignore/skip`

#### App/Addon Name
There will be no default for this question. The user must enter a response. 

#### App/Addon Language

- the default would be `en-US`
- if your system has a different language set (which can be confirmed by using the `echo $LANG` command in the terminal), then _that_ language would be shown as the second option
- if your system was already set to the default language, the second option would not be shown. 
- for "manually define a language", we will validate the response against the allowed language codes
- we have not added a "skip" option here, because the purpose is to focus on improved accessibility in Ember apps.

#### Package Manager

- ignore/skip has been added to cover the use case where the developer is in a workspace 

#### CI Option

Separate RFCs should further define more options for CI. 
  
### Current Limitations
Explicitly, the goal of this RFC is to only include `ember-cli` flags that already exist. Any other possibilities should be considered future work.

This interactive workflow will also ignore any ~/.ember-cli settings that already exist. A separate RFC should be written that makes it possible for these to things to co-exist and respect the settings of the other.

### Future possibilities
An example of a future RFC that could be specific to addons, would be to add a `--supported-version` flag for addons and add that to the wizard: 

- `What is the earliest version of Ember that you intend to support? (use arrow keys)`
  - `> recommended (last two LTS)`
  - `last LTS`
  - `manually define a version number`

### Implementation
This section is not normative, but provided for extra context for folks who might be new to this idea in general. We could use something like [inquirer.js](https://github.com/SBoudrias/Inquirer.js) to implement this wizard, if the mechanism doesn't already exist within the ember-cli codebase. However, it should be explicitly noted that inquirer is only being used as an example of a library that could be used, and this RFC isn't explicitly defining that inquirer.js should be or will be used. 

### Versioning and Stability Statement
Because the `ember new` command only affects new apps, it is not subject to the same semver guarantees that official Ember.js framework libraries currently follow. 

The `ember new` interactive workflow SHOULD NOT be used in other scripts; it is the intent of the design to make this workflow flexible and changeable over time. As such, it should not be considered stable enough to be integrated into other automated tooling. Users should continue to make use of the commands available via `ember-cli` for any other integration scripts. 

## How we teach this

As this is a wholly new idea, it should be documented and added to the guides along with screenshots of the new workflow. Until that prototype exists, this section will largely remain empty. However, we intend to explain it similarly to Vue's documentation for a similar feature (https://cli.vuejs.org/guide/creating-a-project.html#vue-create).  

## Drawbacks

It could be considered a drawback to make this not the default for new Ember apps, even if an app name is defined. 

The general drawback with wizards is that it can lead to a large number of combinations, which increases the risk for fragmentation. We believe that we have limited this risk by limiting the number of questions asked, but it is still a risk and should be identified as such.

## Alternatives

1. We could decide on a different list of questions. See the notes from the [strike team discussion on May 20th](https://github.com/ember-a11y/core-notes/blob/ember-a11y/ember-a11y/2020-05/may-20.md) to review the other possibilities considered.
1. We could make this the default for new Ember apps and add it in the appropriate version as a change.
1. We could have this workflow be opt-in only, via the `--interactive` flag (e.g., `ember new --interactive`). Using the `ember new` command would error and trigger the help workflow (which would be updated to include the `--interactive` flag).
1. We could set the default language to US English (the default language of the Ember project) and remove the language question from the interactive workflow.
1. We could set the default language to US English (the default language of the Ember project) and not have an interactive option at all.

## Unresolved questions

As we identify additional unresolved questions, we will add them here. 
