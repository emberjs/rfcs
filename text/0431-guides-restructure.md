---
Start Date: 2018-01-13
Relevant Team(s): Learning
RFC PR: https://github.com/emberjs/rfcs/pull/431
Tracking: https://github.com/emberjs/rfc-tracking/issues/29

---

# Restructuring the Guides Table of Contents

## Summary

As our favorite framework has grown and changed a lot over the past few years, so have our [Ember.js Guides](https://guides.emberjs.com)! This project aims to use the excellent work that has been done by hundreds of contributors and arrange it in a way that provides a natural learning flow for today’s Ember.js developers.

## Motivation

With Octane coming down the line, and an influx of fresh attention, it’s more important than ever to consider Ember's overall learning experience. It's time for a shift. The current structure and flow of the guides reflects the past, not the present experience for Ember. It’s time to fix that.

When the Guides were originally written, things like object-oriented programming in JavaScript, components, routing, and Single-Page Applications ([SPA](https://en.wikipedia.org/wiki/Single-page_application)) in general were new ideas. For example, they start by teaching the “Ember Objects” programming concept, necessary background information at the time. It would have been an unfamiliar pattern at the time. Now, the learning hierarchy needs have shifted from “learn these unique web development concepts” to “learn how familiar pieces fit together so you can build great apps quickly.”

## Detailed design

The focus of this RFC is not to write new content, but make better use of what is already there by presenting it in a different order. We aim to rearrange the table of contents without breaking any URLs, with refactors necessary to fix the transitions between topics, and with the minimal _new_ content needed to teach Octane. We take care to not make it look like there is “more to learn.”

The overall learning strategy is to establish a common core of sequential knowledge, and then later topics can be read standalone, skipping around. What this means is that if beginners read through to routes, they could skip around the topics lower down the chain successfully.

Finally, it is important to acknowlegde that the Guides are intended to represent the "Happy Path" of using Ember. It is not possible to have something that is a perfect fit for everyone's app needs, nor should they cover our entire API surface.

### Process and preparation

This Table of Contents plan was developed over the course of many months, meetings, and writing sessions. It includes input from many people in the community. Here are the sources of inspiration, information, and planning:

- Chris Garrett's ([@pzuraq](https://github.com/pzuraq)) 2018 Roadmap Blog Post, [Ember as a Component-Service Framework](https://medium.com/@pzuraq/emberjs-2018-ember-as-a-component-service-framework-2e49492734f1)
- My own Roadmap blog post reflections, [Be loud and be ready](https://gist.github.com/jenweber/a9fbea98478fc3841fb8b24f7dc961c8)
- Extensive discussion in Ember Learning Team Meetings, which are held weekly and open to the public
- The [2018 Roadmap RFC](https://github.com/emberjs/rfcs/blob/26c4d83fb66568e1087a05818fb39a307ebf8da8/text/0000-roadmap-2018.md) by [Tom Dale](https://github.com/tomdale)
- The Octane Strike Team meetings and followup conversations
- Hands-on writing of a mini guide for Octane
- Ember.js Framework Core Team meetings
- A [Twitter poll](https://twitter.com/jwwweber/status/1081352702083452928), "Name two things you think the guides should do well/better"
- Discussion on Discord of the initial Table of Contents draft with some key contributors and content creators
- [Dan Monroe](https://github.com/cah-danmonroe)'s trailblazing work to [convert the Guides to use Angle Brackets](https://github.com/ember-learn/guides-source/pull/357)
- Countless hours of Q&A on Stack Overflow and Discord
- Experience training new Ember developers in-house
- Public Ember 101 workshops run in Boston, Massachusetts, USA
- Studying the guides and tutorials of other JavaScript libraries
- The successful restructure and rewrite of the [Ember CLI Guides](https://cli.emberjs.com) (a far more drastic project than this RFC aims to be)
- Attempting to reorder the Guides in a branch, as an experiment

We are confident that this is possible thanks to the incredible response and effort shared by the community for the CLI Guides work.

### Audience

There are many different types of documentation within Ember, so it's important to identify a target audience for each, in order to guide decision making and provide a learning flow that grows with the reader.

There are 3 main audiences for our resources: 

1. newcomers
2. upgraders
3. specific-answer-seekers

A huge challenge within the Guides is that they somewhat serve all three, and we'd like to have cleaner distinctions between sections and resources. For example, the Quickstart and Tutorials are distinct resources from the rest of the guides, and they clearly target new users. Upgraders could be served by adding 1 section to the guides as they are today. Specific-answer-seekers must lean on both the API docs and the Guides to find what they need.

We propose that the Guides should focus on the following audience - newcomers who have done the tutorial, and want to build real apps. It will show the "happy path" and how the parts of Ember interact to form good, useful patterns, in a way that would be out of scope for an API documentation block.

Along these lines, anything that can be offloaded to the API docs, should. For example, the Guides should show using a Computed Property/tracked property to solve a problem in a Component or Controller. They should not get into the depths of syntax option for one aspect of an API surface, but rather link to them.

We also have the current hypothesis to inform intro sections, as suggested by [Mike North](https://github.com/mike-north): "People want to get the smallest understandable atom that they can jump in on." This will help us refactor how we present different topics.

### Table of Contents

The following Table of Contents will be applied iteratively over the course of many months. It is not considered a blocker for the release of the Octane edition. The quick wins and urgent Octane refactors will be applied ASAP, and are described in the Implementation section of this RFC. Justification for the ordering is provided following the topics list.

**Core Concepts**

- What is Ember?
- Getting Started
- Anatomy of an app (future new content)

**Fundamentals**

- Templating
- Working with JavaScript
- Components
- Routing (includes Routes, Router, and Controllers)
- State management (aka computed properties)
- Services

**Leveling up**

- Data management (future new content that references separate Ember Data guides)
- Addons and dependencies
- Testing
- Configuration
- Deploying
- Upgrading (will contain links to upgrade resources, Editions, Deprecations)
- Developer Tools (currently named Ember Inspector)
- Reference 
    - Accessibility
    - Syntax conversion guide

### Topic naming

As much as possible, we aim for the topics to be named after their general web development concepts, and not the Ember-specific implementation of them.

### Logic for ordering and grouping

In this section, we will cover why this order and grouping is being proposed.

#### Groupings

Groupings are added for the benefit of new learners. They break up the content visually and gives them a clue about which sections to pay the most attention to.

Just because something is included in "Leveling up" does not automatically make it an "advanced" topic. The main logic for the division between "Fundamentals" and "Leveling up" are the following tests: Can someone ignore this section and still use Ember effectively? Can it be learned in any order if someone knows the basics? If so, it goes in "Leveling up."

As we look at each topic, we also ask: Does someone need prior knowledge of another topic in order to understand this section? Whatever those "prior knowledge" topics are, we add them as Fundamentals.

#### Anatomy of an app

The most common points of confusion for new devs have become things like “how does everything fit together? Why can't I call a sibling component's actions directly?” The faster we can get the “Mental Model” of Ember across, the better. The Super Rentals tutorial covers the bases well for using the CLI and adding basic interactions. Anecdotally, thanks to the Tutorial, it seems that the missing pieces in the guides are mostly architectural. "Anatomy of an app" is the main area of new content, where we attempt to give an overview of the topics a web developer has to care about, the parts of Ember that address them, and the file structure. Current Ember developers will reference this section while learning new features.

#### Templates & Template helpers

Templates come before components, since it is possible for someone to not know JavaScript/Handlebars and still contribute to Ember apps. For example, designers could effectively work in only templates, using plain HTML. To make Templates work as the opening topic, additions are necessary in the introduction.

Template Syntax will cover Ember Handlebars basics and built-in features like `{{#each}}` and `{{#if}}`. It will take care to make it clear that Ember uses its own implementation of Handlebars. It will clearly differentiate between control structures (loops and conditionals), inline vs block styles, component invocation, and helpers.

Template helpers are split out and moved to "Leveling Up" because developers do not need to use them to write basic Ember apps (they are not a prerequisite). This section includes writing custom Helpers and using built-in helpers like `get`, `let`, and `array`. Although helpers themselves are not considered a difficult topic, they are used most often when developers understand their app's architecture and the flow of data. Also, template helpers have more in common with JavaScript functions than concerns relating to layout and data flow.

Lastly, another reason to split out Template syntax from Template Helpers is because the syntax section is expected to expand with the addition of Octane content.

An alternative to this split is to reduce the documentation of helpers themselves in the Guides, and lean on the API docs instead, however great care should be taken in removing documentation. Without a strong case for it, we lean towards leaving them in the Guides.

#### Working with JavaScript

A common concern of new learners is, what is JavaScript and what is special to Ember? The Working with JavaScript section incorporates the existing JavaScript Primer and new content that will help make the distinctions clearer. Ember.js has proactively adopted JavaScript APIs as they become available, such as Classes and Decorators, in some cases before the larger JavaScript community becomes familiar with them. Key people in the Ember Community are also involved in the development of JavaScript itself through TC39. To an extent, it is our responsibility to at least suggest to developers which JavaScript concepts they must learn in order to feel comfortable with Ember.

#### Components

Components will need the most refactor work and new content for Octane, so here’s the subtopic breakdown:

- Creating a component (includes Component Types post-Octane)
- Displaying data in a Template
- Adding actions
- Nesting components (includes suggestions of when to break things into components)
- Passing arguments
- Using computed properties (extremely basic example, link to the Computed Property section)
- Working with arrays of data
- Lifecycle hooks

#### State management

The State Management section is important because whether "computed properties" are available in their current state or as `@tracked` Decorators, the nuances must be within easy reach for existing Ember developers and future learners. We rename this topic in alignment with the Topic Naming convention mentioned above.

#### Routing

This RFC proposes grouping Routes, Routing, and Controllers under the same topic heading.

If that sounds like a lot, it's because it is. "Routing" is a responsibility that is divided into many pieces:

- the route declarations in `router.js`, including dynamic segments
- the Handlebars route template, the middleman for passing data from the controller to components
- the Route JavaScript file, which contains the model hook for fetching data and Transition rules in the form of hooks. The model hook receives the query params that are defined in the Controller
- Controllers, which hold the results of the model hook, actions and other attributes that need to be passed to child components, and query param definitions.

Common beginner mistakes are to define actions and attributes in a Route and try to pass them to components, plus attempts to access the result of the model hook in functions in the route JavaScript. Beginners think of Routes as "special components" and are surprised by their limitations. This becomes a long-lived pain point if developers hand off the controller's responsibilities to Components.

For better or for worse, there's a mental-model codependency between Routes and Controllers that is unlikely to change in the near term. Our Guides should reflect the best possible learning experience for today's constraints, rather than be a reflection of the codebase architecture. By grouping these topics together, we have an opportunity to heal confusion over Controllers and help new developers avoid unexpected pitfalls.

Based on initial feedback, this grouping is the most divisive part of the proposal. This grouping is informed by reflecting on how one might teach Controllers to a new developer, while working together in person. In order to reach consensus, any objections to the grouping should suggest solutions that include an explanation of how and where one would teach Routes, actions, routing with Query params, and passing arguments to the model hook.

#### Services

The Services documentation is currently sparse, but it is included in "Fundamentals" for two reasons. First, an understanding of Services is a prerequisite for understanding Ember Data's store service. Second, the Router service is not well known by new developers, as it is solely found in the API documentation, and it provides behavior that new developers expect to have. Third, the popularity of the mental model of Ember as a Components-Services framework is a signal that this may be an effective teaching strategy.

#### Ember Data

Ember Data will be gradually extracted into its own Guides. 2-3 years ago, there was a push to provide better official documentation of Ember Data, as a "first class citizen." It was intermingled with the rest of the Ember.js Guides as a result. We will continue to treat Ember Data as a first class citizen, yet with improved separation of concerns similar to the approach to Ember CLI Guides. Ultimately, the content in other Guides topics will be refactored to show both Ember Data and non-Ember Data approaches, in an effort to lower the perceived cognitive overhead of Ember.

This change reflects the overall drive of the JavaScript Ecosystem towards interchangeable, composable parts.

The Ember Data team is especially requested to review this shift and provide feedback.

#### Deploying and Upgrading

These sections have some existing content. They will aim to _not_ duplicate the contents of the CLI Guides, but rather reference content found there. They are included in the top-level Table of Contents for two readons, discoverability and to showcase Ember's strengths

#### Ember Inspector

Although the Ember.js Guides are versioned, the Inspector Guides do not need to follow the same versioning strategy. It would be reasonable to separate them out into their own, unversioned Guides. However the content is quite stable and therefore lower priority than other refactors.

The Ember Inspector team is requested to consider whether naming the section "Debugging" would improve discoverability of Ember Inspector for new developers, and whether the eventual unversioned separation aligns with expected technical development.

#### Unchanged topics

Configuration, Testing, and Addons & Dependencies remain unchanged in their approach. There are plans underway to make the Tutorial an unversioned, separate resource. That is outside the scope of this RFC, however this Table of Contents assumes that work comes to completion.

#### Removals

Notably missing is "Ember Object Model." This is on purpose. It will be pulled into other sections, in a “show, don’t tell” kind of approach. Also removed is "Application Concerns," which are separated into their appropriate alternate subtopics.

### Why doesn’t this RFC include rewriting content?

Individual pages have already been refactored over the past two years by many contributors. Examples include the Ember Objects page, Controllers, using third party libraries, and explanations of data management. Many of the pain points that current Ember devs remember from their early days have been fixed. For example, it’s clear that Ember Data/JSONAPI aren’t mandatory, that you *can* use things like fetch, that Computed Properties need to be consumed for them to fire… we’re in a pretty good place! Those improvements of individual topics may continue without the need for an RFC.

If we choose a good structure for the Table of Contents, it will make it much easier to write/rewrite individual sections.

It is also important to note here that that Ember does not have a foundation or funding arm, so although it would be great to have a dedicated writer tackle this from the ground up, we must choose an approach that would be realistic for a group of volunteers to achieve.

### Technical approach

Thanks to Chris Manson’s work ([@real_ate aka @mansona](https://github.com/mansona)) on the Guides app architecture, we can move content around while preserving existing links! The Table of Contents specified in the `pages.yml` file of guides-source can have any paths, and is not 100% dependent on the physical file structure to create URLs. It is very important that we don’t break existing blog articles, community tutorials, Stack Overflow answers, etc, both for user experience and SEO reasons.

All guides content is markdown. When we rearrange content, we’ll have to change some links and add redirects. However there are tests in place that check for bad links, so we can do this confidently.

## How we teach this

Community buy-in is important to reduce perception of churn and make “leveling up” our resources a team effort. Significant attention will be put towards informing the community of upcoming changes, and giving them the opportunity to participate.

We will also test major changes with beginner-level developers and developers who don’t know Ember. Although the rollout for the live site will take months, a rough cut, undeployed, could be completed in 1-2 weeks. It could serve as a North Star for the work to be done.

### Implementation plan

This work will need to be done incrementally over many months/the next year. It will be communicated in the form of Quest issues, with help requested via the Ember Times, Discord, Discuss, etc.

Community members will be asked to participate in PR reviews. A diversity of technical experience levels, language backgrounds, and use case perspectives will create a stronger output.

The intial steps will aim for the quick wins and the urgent changes that aid in Octane documentation.

Here's what we could expect a minimal first pass to look like:

- What is Ember?
- Getting Started
- Tutorial (already in progress of being split out)
- Templates
- Components
- Routing (includes Routes, Router, and Controllers)
- State management (aka computed properties)
- Services
- Ember Data (renamed from "Models")
- Addons and dependencies
- Testing
- Configuration
- Application Concerns
- Ember Inspector

The groupings like "Fundamentals" require architectural work on the guides app. They can be done in parallel depending on volunteer capacity and interest, or delayed until the end.

## Drawbacks

This refactor is biased towards new user experience, so existing Ember users could experience the most drawbacks.

1. They will need to discover where old content lives. This is mitigated by the site search, which is now stabilized
2. Experienced developers who haven’t looked at the Guides for a long time will all reference it during the Octane upgrade, and may be surprised to find a new layout. It’s another “new thing to learn.” This is mitigated by consolidating most of the new things to learn into the Components section
3. There will likely be some wrinkles to iron out with regards to content that should have been refactored during rearranging, but was overlooked. We are confident that the community will help identify these issues.

## Alternatives

The main architectural pattern choices were as follows:

1. On one extreme, make every section standalone
2. The middle ground, establish a “Common Core” to be read sequentially, and then have standalone sections that rely on knowledge of the core
3. The other extreme, make the entire guides sequential

We choose the middle ground, because it requires the least new writing, and if we chose to move towards one of the other extremes, it does not prevent that choice nor would we throw away work.

It’s also useful to study the learning flow of other front end libraries in order to determine possible alternatives. Let’s look at a few.

### React

React is known for having low learning overhead for someone who is making their first app. With its popularity, we can guess that new users may expect to find similar topics easily accessible in our guides. This list is most useful for considering what should be in our Components section.

1. Hello World
2. Introducing JSX
3. Rendering Elements
4. Components and Props
5. State and Lifecycle
6. Handling Events
7. Conditional Rendering
8. Lists and Keys
9. Forms
    1. Lifting State Up
    2. Composition vs Inheritance
    3. Thinking In React

The Overview section of the React Tutorial also helps show what we may be missing:

- What Is React?
- Inspecting the Starter Code
- Passing Data Through Props
- Making an Interactive Component
- Developer Tools

Nowhere on our current site do we have a highly visible explanation of what Ember is, beyond snippets. In light of this glaring omission, we have added a “What is Ember” section to the Guides Table of Contents above. It is not meant to replace the ongoing “Why Ember” and marketing-focused descriptions that are underway.

### Vue

As a fully-featured framework, Vue is an easier comparison for possible Table of Contents listings. Keep in mind that much of this type of content is present in our CLI docs instead, so this list will look longer than what we are aiming for.

- Introduction
- What is Vue.js?
- Getting Started
- Declarative Rendering
- Conditionals and Loops
- Handling User Input
- Composing with Components
- Relation to Custom Elements
- Ready for More?
- The Vue Instance
- Template Syntax
- Computed Properties and Watchers
- Class and Style Bindings
- Conditional Rendering
- List Rendering
- Event Handling
- Form Input Bindings
- Components Basics
- Components In-Depth
- Component Registration
- Props
- Custom Events
- Slots
- Dynamic & Async Components
- Handling Edge Cases
- Transitions & Animation
- Enter/Leave & List Transitions
- State Transitions
- Reusability & Composition
- Mixins
- Custom Directives
- Render Functions & JSX
- Plugins
- Filters
- Tooling
- Single File Components
- Unit Testing
- TypeScript Support
- Production Deployment
- Scaling Up
- Routing
- State Management
- Server-Side Rendering
- Internals
- Reactivity in Depth
- Migrating
- Migration from Vue 1.x
- Migration from Vue Router 0.7.x
- Migration from Vuex 0.6.x to 1.0
- Meta
- Comparison with Other Frameworks
- Join the Vue.js Community!
- Meet the Team

One possible lesson here is that we could split up Components like Vue did with Component Basics and Components In-Depth. Their dedicated section on Computed Properties inspired the inclusion in our new Table of Contents.

### Angular

Angular is also a full-featured framework that has a lot in common with Ember.

FUNDAMENTALS
- Architecture
    - Architecture Overview
    - Intro to Modules
    - Intro to Components
    - Intro to Services and DI
    - Next Steps
- Components & Templates
    - Displaying Data
    - Template Syntax
    - User Input
    - Lifecycle Hooks
    - Component Interaction
    - Component Styles
    - Angular Elements
    - Dynamic Components
    - Attribute Directives
    - Structural Directives
    - Pipes
- Forms
- Observables & RxJS
- Bootstrapping
- NgModules
- Dependency Injection
- HttpClient
- Routing & Navigation
- Animations

SETUP & DEPLOYMENT

- Project File Structure
- Workspace Configuration
- npm Dependencies
- TypeScript Configuration
- Ahead-of-Time Compilation
- Building & Serving
- Testing
- Deployment
- Browser Support
- Dev Tool Integration

Angular is the closest match to our current guides structure. Notably, they work Styles in as part of their Components section.

### Trends

All three of the libraries above cover forms in their own dedicated section. They also cover styles and animation, which we do not cover at all. These are all good candidates for future guides.

## Unresolved questions

- What does the community think of this structure? How can it be improved?
- Is it good for new learners? Is it good for existing users?
- What possible pain points does the community see?
- Are there any areas missing from the Table of Contents?
- What do people think of removing the “Ember Object Model” section?
- Are the goals of the Ember Data and Ember Inspector teams supported by this new layout?

### One last note

The Guides are one of those things where everyone has an opinion, and that's ok! However, as a reminder, please be kind and constructive in your comments. The Guides and the tools they cover are the work of many dedicated community members. They are authored and maintained through donated effort, both by unpaid individuals and companies who encourage their teams to contribute. Although we always know there is room for improvement, we are proud of where we came from and excited for where we're going next!
