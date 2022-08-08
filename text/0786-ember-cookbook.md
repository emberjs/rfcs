---
stage: accepted
start-date: 2021-01-11T00:00:00.000Z
release-date:
release-versions:
teams:
  - learning
prs:
  accepted: https://github.com/emberjs/rfcs/pull/786
project-link:
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

# Ember cookbook proposal

## Summary

Adding a cookbook section to our learning resources will help Ember developers to learn maintainable, accessible patterns for common tasks.

## Motivation

Ember docs in its current state are missing [how-to guides](https://documentation.divio.com/how-to-guides/) that would provide goal-oriented answers / patterns to common problems Ember developers face on a daily basis. For example, if someone wants to learn how to handle a form submission, they would have to look at blog posts.
Additionally, many Ember users over the years have requested a cookbook-like resource. Early versions of Ember had a [resource like this](https://guides.emberjs.com/v1.12.0/cookbook/), but it was removed in 1.13.


## Detailed design

### Creation
Ember-cookbook will be a repo under [ember learn](https://github.com/ember-learn). We start by introducing an Ember Cookbook section in the ember docs dropdown.

Currently, there is a lot of material out there in the form of blogs, stackoverflow answers and efforts like Ember Atlas. The challenge is to make sure they are discoverable.
An official Ember cookbook will be a great way to officially endorse patterns, since not all patterns found on all blogs can be trusted.

Sourcing information for the cookbook can be done a few ways:
Officially endorsing existing articles and blogs
Monitoring most asked questions on stackoverflow
Create entries for [really good explanations](https://discuss.emberjs.com/t/adding-a-delete-row-button-from-a-table/18623) from the ember forum.
Create sections that would make it easier for the developer to navigate through the different guides. Ember-data's adapter/serializer cookbook articles can go under a specific section called "Ember Data" within the cookbook.

For the MVP we can include the following articles and topics, or similar content.
[How to create a grid/table with Ember Data and creating a CRUD form with Ember data](https://discuss.emberjs.com/t/looking-for-an-example-of-a-grid-and-form/18490)
[Show or Hiding content based on current Route](https://discuss.emberjs.com/t/show-or-hiding-content-based-on-current-route/18567)
[Synchronising query parameters with a component](https://discuss.emberjs.com/t/synchronising-query-parameters-with-a-component/18084)
[Ember without Ember Data](https://stackoverflow.com/questions/24408892/ember-without-ember-data)
[Updating entries from previous cookbook](https://guides.emberjs.com/v1.12.0/cookbook/)

### Contributing & Maintaining
The Ember learn team will manage this repository and will guide what goes into the cookbook.
In addition, we could have a set of volunteers sign up to add content to the cookbook regularly. Some of the tasks to maintain the cookbook would include,
Monitoring stack overflow and other forums to come up with most-asked questions
Work closely with the Ember Core team on upgrades to add, modify or delete information from Ember Guides.

A cookbook template will be used to keep all the entries uniform. The template would contain the following sections,

#### Title
  This is an intro section that lets someone know what they will learn by the time they are done reading.
#### The challenge
  Lay out the example and the problem you are trying to solve
#### Steps
  The steps someone should take to get to the end result.
  Use headings to describe steps and not numbers.
#### Results
  When you are finished, here's how it should work
#### Resources
  Links to Guides and API go here.
  Also include ways that someone could build upon this knowledge.
#### Tags

After reading this recipe you should now know what is a cookbook and what is the templete that Ember recipes follow.

#### Learn more

### Launch strategy
The cookbook articles will be drafted in markdown format. Once there is a critical mass of 5 articles, they can be published. The cookbook will be hosted at cookbook.emberjs.com
The cookbook will use Guidemaker, similar to cli.emberjs.com
We will mention it in the Ember Newsletter.
To Evangelize it further, we can share the links on social media and as answers to stackoverflow questions.

### Versioning
We can show a single version of the recipe and have a “last updated at” and a field that shows the range of versions that it is applicable to. This would make it easier to maintain.


What kinds of content belong in the Cookbook?
One challenge for the cookbook is determining where content belongs.

Today, Ember has the following key learning resources:
The Super Rentals Tutorial
The Ember Guides
The CLI Guides
The API docs
Community-maintained learning resources like blogs, videos, and livestreams

We know that something may belong in the cookbook if we answer yes to the following questions:

- The concept can be explained in an article that would take less than 10 mins to read
- The concept is not already shown in the Super Rentals tutorial
- The article helps show how to put together multiple Ember features to achieve a goal.
- The article is trying to demonstrate a pattern, not teach a concept. For example, if we are showing how to use a model hook to accomplish something, we are not explaining what a model hook is.
- [It is narrowly tailored to solve a single problem](https://guides.emberjs.com/v1.12.0/cookbook/contributing/deciding_if_a_recipe_is_a_good_fit/#toc_solution)
- It shows how to solve a problem that Ember apps may commonly face. For example, managing a dropdown menu or creating a form are common. Integrating WebRTC is not.
- The article goal can be accomplished without installing new addons, with the exception of addons mentioned in the Guides.

Removal of recipes
If a topic becomes obsolete for any reason, we could add a warning at the top. An archived section for outdated things would help with this. This way the links stay forever but hidden in a section. This way it will serve as an example of which pattern is deprecated.

## Drawbacks

We will need to maintain the cookbooks as we have major Ember version upgrades.

### Language support

The Ember documentation serves a global community. The cookbook aims to be a resource that could be translated into multiple languages after the initial content settles. The translations would be found under the same domain, `cookbook.emberjs.com`


