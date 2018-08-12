- Start Date: 2018-07-30
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Ember CLI Docs

## Summary

This RFC proposes converting the existing [Ember CLI website](https://ember-cli.com/) into an
Ember app, restructuring the table of contents, replacing a significant
portion of the learning material, and inviting community members to participate in writing new content.

## Motivation

The purpose of these changes are to empower new contributors, create a consistent
narrative structure, correct outdated information, and lead new readers through an easier learning progression.

Ember's public sites are being migrated from Ruby apps to Ember apps
in order to improve maintainability and empower new contributors. The CLI docs
are currently a Jekyll app. Similar migrations have been very successful.

The rewrite and/or reorganization of content is driven by an audit of the existing
content's relevance and balance. While trying to plan a refactor in place, it became clear that a greenfield approach is more time efficient and will lead to a 
better learning experience. A significant portion of the content from the current guides site can be ported over once a new structure is in place.

## Detailed design

This app will have a new table of contents. The architecture will follow the
same patterns successfully used in other apps that have been converted from
Middleman apps to Ember.

### Writing process

Writing new content and porting over existing information is a job that will require the help of many contributors! After this RFC is accepted, a call for contributors will be made. Each unwritten section will have comments indicating which topics to cover. In cases where content has been ported over, comments will indicate which sections to fact-check, clarify, or revise. Following a verson one release, writing work will be organized via normal GitHub issues.

Since maintaining consistent voice and structure across a blank slate is a challenge, beta content for the core learning experience has already been drafted, including Basic Use guides and a tutorial for creating an addon from start to finish.

The beta version of the CLI Guides content can be found at [ember-learn/cli-guides-source](https://github.com/ember-learn/cli-guides-source). The Markdown files there are rendered by [ember-learn/cli-guides-app](https://github.com/ember-learn/cli-guides-source). The app is currently [deployed on Netlify](https://sad-noyce-026d89.netlify.com/) for testing and UX validation, however it requires more work to resolve many known bugs.

A writing styleguide will be provided for contributors.

### User Personas

The content layout should follow the progression of an Ember developer's
learning experience. There are four main user personas for the 
CLI documentation:

1. A new or "typical" Ember CLI user - someone whose primary work is
running common commands like `ember serve` and who has a "zero
config" type of experience with Ember
2. Power users - developers who make their own configurations to the
build pipeline
3. Beginner addon authors - those who are looking to build simple
shared UI components, methods, or wrappers for existing npm libraries
4. Advanced addon authors - those who dig into internals to make their
addon work, or who are planning for broad extensibility

### Table of Contents

Applying these User Personas to the CLI content, the following topics layout emerges. "Beginner" topics will include links to later "Advanced"
topics, similar to how the Guides link to the API docs.

- Introduction 
    - how to install ember cli 
    - a very simple, short definition of what it is (the official way to create, build, and test an Ember app)
    - Why is the CLI needed
    - Guidance on learning path
    - How to contribute
- Basic use (explain options of each)
    - CLI Commands: Explain how to use the `help` command and common commands like `ember new`, `ember server`, `ember generate`, etc. Each is explained briefly, together with an example usage and a link to the Main Ember Guides with more information about how to use those files.
    - How to find and use addons
    - How to use npm packages
    - Installation and Upgrading the CLI (including a note about upgrading your app, with a link to more resources)
    - feature flags & configurations
- Advanced use
    - shims
    - broccoli
    - custom blueprints
    - CSS compilation
    - Using another testing library
    - more on dependencies
    - more configurations
- Writing Addons
    - Overview
    - Tutorial: Creating a standalone addon and an in-repo addon, 
    - Using the dummy app
    - Including assets
    - Testing your addon
- API Documentation
    - brief description of the target audience and a link

### Versioning

Only one version of the documentation will be deployed and maintained.
The documentation app itself will have clear releases
as major changes are made, so that users working on
older apps can still go back in time if they need to.

The url will contain `/release/` so that if versioning is needed in the future,
the option is available.

### Transition and legacy links

While the project is in development, it will be worked on as a separate site, and the main site, [https://ember-cli.com](https://ember-cli.com) will remain in place.

Legacy links should be maintained because deprecating the links would cause SEO problems. The options include:
- Leaving the old guides as-is, with deprecation banners
- Finding a way to replicate the URL structure of the current guides site within the new site, and linking to deprecated content as needed.

For SEO reasons, the second option is preferred.

Upon reaching feature parity, [https://ember-cli.com](https://ember-cli.com) will receive the new content. Between the first release and feature parity with the old site, content will be hosted at [https://guides.emberjs.com/cli].

### Application architecture

The application architecture will follow similar patterns as other Middleman
apps that have been successfully turned into Ember apps. Some examples of past conversions are:

- [Deprecations](https://github.com/ember-learn/deprecation-app)
- [The Guides](https://github.com/ember-learn/guides-app)
- [The API docs](https://github.com/ember-learn/ember-api-docs)

[Chris Manson](https://github.com/mansona?tab=overview&from=2018-06-01&to=2018-06-30) has a project in development that automates the creation of documentation apps, integrating the lessons learned from these past conversions. Early results are looking great!

The resulting app will make use of typography and UI assets from 
[ember-styleguide](https://github.com/ember-learn/ember-styleguide)

Although only one version will be deployed/maintained for the forseeable future, the URL structure will allow for future growth, i.e. `https://guides.emberjs.com/cli/release/some-topic`

### Maintaining content

With module unification and tree shaking refactors underway, there may be some big changes to Ember's
file structure. There are a few ways to mitigate this, while still maintaining only one version of these guides:

1. Whenever possible, the CLI guides should link to the Ember Guides. The details of file layout and syntax are best handled in a resource that is versioned.
2. The CLI guides can also frequently give a nod to past configurations/features. A url checker will make sure that these "legacy" resource links still exist. The pace of major version releases is slow enough that this should be sustainable.
3. As mentioned earlier, the urls for the cli guides will include `/release/` in case future versioning is needed

Members of both the Learning Core Team and Ember CLI Core team will have merge access.

## How we teach this

Overall, bringing the CLI docs content up to speed and making it
more maintainable should result in better integration of the
CLI documentation into the Guides. The current content is out
of date, and so it is not frequently linked.

The impact to new users will be a better experience. Existing
Ember users may have an adjustment period to learn the new layout,
but the current layout is confusing, so we believe there will be
net improvement from day one. The addition of search tools will help
with the transition.

Links in the Guides will need to be updated to point
to the new documentation app. There are 41 links to the
current ember-cli website, but only a handful are unique.

The Ember CLI website is not referenced in the API docs.

## Drawbacks

Some potential drawbacks include:

- Old bookmarks will still point to old content, and it is significant engineering effort to maintain those legacy links
- Users may be used to finding content in a particular place
- Some existing content will be deemphasized or removed
- It's another app to keep in step with the main website

## Alternatives

An alternative is to refactor the content in place. This will be more
time consuming, and will not achieve a consistent narrative voice or cumulative learning experience.

## Unresolved questions

- How should legacy links be handled from a technical perspective?
- The Table of Contents topic list is not yet complete
- Advanced topics require further input from experienced addon authors
