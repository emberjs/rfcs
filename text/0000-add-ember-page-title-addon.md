- Start Date: 2020-06-24
- Relevant Team(s): Ember CLI
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)
- Authors: Benjamin Jegard

# RFC to add ember-page-title to the default app blueprint

## Summary

Add https://github.com/adopted-ember-addons/ember-page-title as a default addon for the app blueprints. (addons too ?)

## Motivation

This RFC is part of the work made by Ember.js Accessibility Strike Team to ensure that newly created ember apps have no accessibility issues.

Users with assistive technology rely on the page title to know if they are on the correct page of a website.
Adding this addon will provide developers a simple solution to achieve the [WCAG Success Criterion 2.4.2: Page Titled](https://www.w3.org/WAI/WCAG21/Understanding/page-titled.html)

## Detailed design

1. Move ember-page-title to the ember-cli org (?) (already in ember-adopted-addon org)
2. Add the dependency to the app blueprint here: https://github.com/ember-cli/ember-cli/blob/master/blueprints/app/files/package.json#L19
3. Update app blueprint to include `<HeadLayout />` on top of https://github.com/ember-cli/ember-cli/blob/master/blueprints/app/files/app/templates/application.hbs
4. Update route blueprint to include `{{page-title "Route name"}}` on top of the route template file

## How we teach this

Update the "Page Title" section in [Page Template Considerations](https://guides.emberjs.com/release/accessibility/page-template-considerations) to use `ember-page-title`.

## Drawbacks

- An additional dependency.

## Alternatives

## Unresolved questions

None

## References

- [WCAG Success Criterion 2.4.2: Page Titled](https://www.w3.org/WAI/WCAG21/Understanding/page-titled.html)
