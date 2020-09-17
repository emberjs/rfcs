- Start Date: 2020-06-24
- Relevant Team(s): Ember CLI
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)
- Authors: Benjamin Jegard, Melanie Sumner, Ricardo Mendes

# Add ember-page-title to the app blueprint

## Summary

Add [`ember-page-title`](https://github.com/adopted-ember-addons/ember-page-title) to the default blueprint for new Ember apps as a way to provide improved out-of-the-box (OOB) accessibility for Ember applications.

## Motivation

This RFC is part of the work made by the Ember.js Accessibility Strike Team to ensure that newly created ember apps have no accessibility issues.

Users with assistive technology rely on the page title to know if they are on the correct page of a website.
Adding this addon will provide developers a simple solution to achieve the [WCAG Success Criterion 2.4.2: Page Titled](https://www.w3.org/WAI/WCAG21/Understanding/page-titled.html)

## Detailed design

1. Make [`ember-page-title`](https://github.com/adopted-ember-addons/ember-page-title) an official Ember addon by transferring `ember-page-title` repo to the [Ember CLI org](https://github.com/ember-cli) (it's currently in the [Adopted Ember Addons org](https://github.com/adopted-ember-addons))
2. Add the dependency to the app blueprint here: https://github.com/ember-cli/ember-cli/blob/master/blueprints/app/files/package.json#L19
3. Update app blueprint to include `<HeadLayout />` at the top of [application.hbs](https://github.com/ember-cli/ember-cli/blob/master/blueprints/app/files/app/templates/application.hbs) as well as a helpful comment explaining what it is used for
4. Update route blueprint to include `{{page-title RouteName}}` at the top of the route [template.hbs](https://github.com/emberjs/ember.js/blob/master/blueprints/route/native-files/__root__/__templatepath__/__templatename__.hbs) (this [template.hbs](https://github.com/emberjs/ember.js/blob/master/blueprints/route/files/__root__/__templatepath__/__templatename__.hbs) too) where `RouteName` is the name of the route provided to the `ember generate route` command

## How we teach this

- Update the "Page Title" section in [Page Template Considerations](https://guides.emberjs.com/release/accessibility/page-template-considerations) to use `ember-page-title`.
- Update code examples in [Building Pages](https://guides.emberjs.com/release/tutorial/part-1/building-pages/) to include uses of `{{page-title}}`. Also explain that updating the page title with the current page name will help users with assistive technology locate themself on the website.
- Update [ember-welcome-page](https://github.com/ember-cli/ember-welcome-page) addon and [WelcomePage](https://github.com/ember-cli/ember-welcome-page/blob/master/addon/templates/components/welcome-page.hbs) component to explain the contents of the fresh project a bit better. The component should explain the presence of `{{page-title}}` and link to the "Page Title" section in [Page Template Considerations](https://guides.emberjs.com/release/accessibility/page-template-considerations)

## Drawbacks

- An additional dependency to new projects. But, no noticeable size differences it's only all +10kb of js for production builds (not gzipped, ~2kb js gzipped)
- Add an extra helper `{{page-tile}}` and a new component `<HeadLayout />` (possible name clashing)
- Slightly increases the learning curve. Increases the amount of code in a first-starter project, which could become overwhelming.

## Alternatives

N/A

## Unresolved questions

None

## References

- [WCAG Success Criterion 2.4.2: Page Titled](https://www.w3.org/WAI/WCAG21/Understanding/page-titled.html)
