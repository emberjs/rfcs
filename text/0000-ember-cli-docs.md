- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Ember CLI Docs

## Summary

This RFC proposes converting the existing [Ember CLI website](https://ember-cli.com/) into an
Ember app, restructuring the table of contents, and replacing a significant
portion of the learning material. The new documentation and app will be found
at [https://emberjs.com/cli](https://emberjs.com/cli). Although the old site will be
deprecated, it will remain publicly available with a clear label and pointer 
to the new site.

## Motivation

The purpose of these changes are to empower new contributors, create a consistent
narrative structure, and lead new readers through a reasonable learning progression.

Ember's public sites are being migrated from Middleman apps to Ember apps
in order to improve maintainability and empower new contributors. The CLI docs
are currently a Middleman app. Similar migrations have been very successful.

The rewrite and/or reorganization of content is driven by an audit of the existing
content's relevance and balance. Following an audit fo the content, 
it became clear that a greenfield approach is more time efficient and will lead to a 
better experience for developers who are new to Ember.

## Detailed design

This app will have a new table of contents. The architecture will follow the
same patterns successfully used in other apps that have been converted from
Middleman apps to Ember.

### User Personas

The content layout should follow the progression of an Ember developer's
learning experience. We have identified up to four user personas for the 
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

Applying these User Personas to the CLI content, the following Table of
Contents emerges. "Beginner" topics will include links to later "Advanced"
topics, similar to how the Guides link to the API docs.

- Landing page 
    - how to install ember cli 
    - a very simple, short definition of what it is (the official way to create, build, and test an Ember app)
- Overview
    - What is the CLI
    - Docs layout
    - Why it should be used
    - How to contributoe
- Quickstart Tutorial
    - ember new
    - ember server
    - ember test
    - ember generate
    - ember install (incl some general info about ember addons vs npm packages)
- Basic use (explain options of each)
    - ember new
    - ember server
    - ember test
    - ember generate
    - ember install (incl link to later section on shims for npm packages)
    - feature flags & configurations
    - Environmental variables
    - File tree reference
    - addons/dependencies
    - Upgrading
- Advanced use
    - shims
    - broccoli
    - custom blueprints
    - CSS compilation
    - Using another testing library
    - more on dependencies
    - more configurations
- Addon authoring
    - Overview
    - What is an addon (technically)
    - Learning to build addons
    - Creating a standalone addon
    - Creating an in-repo addon
    - Making an npm package wrapper
    - Using the dummy app
    - Including assets
    - Testing your addon

### Versioning

Only one version of the documentation will be deployed.
The documentation app itself will have clear releases
as major changes are made, so that users working on
older apps can still go back in time if they need to. 

### Handling legacy links

Legacy links will still direct to the old site,
which will have a large deprecation banner on each page. This lets us
maintain existing SEO, links from blog posts, etc. without being
constrained by the old content or trying to engineer fancy redirects,
which are prone to breaking.

### Application architecture

The application architecture will follow the same patterns as other Middleman
apps that have been successfully turned into Ember apps. Some examples are:

- [Deprecations](https://github.com/ember-learn/deprecation-app)
- [The Guides](https://github.com/ember-learn/guides-app)
- [The API docs](https://github.com/ember-learn/ember-api-docs)

It will make use of typography and UI assets from 
[ember-styleguide](https://github.com/ember-learn/ember-styleguide)

### Search
The CLI docs do not currently have search available. By moving the docs
to an Ember app similar to the Guides, we can use the same Algolia approach
from the Guides in order to support internal search.

## How we teach this

Overall, bringing the CLI docs content up to speed and making it
more maintainable should result in better integration of the
CLI documentation into the Guides. The current content is out
of date to the point that it is not advisable to link to specifics
on the site.

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

- Old bookmarks will still point to old content
- Users may be used to finding content in a particular place
- Some existing content will be deemphasized or removed
- It's another app to keep in step with the main website

## Alternatives

An alternative is to refactor the content in place. This will be potentially more
time consuming, and will not achieve a consistent narrative voice or cumulative
learning experience.

## Unresolved questions

???
