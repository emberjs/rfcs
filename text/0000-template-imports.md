- Start Date: 2019-02-14
- Relevant Team(s): Ember.js
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# TODO Template Imports

TODO:

* import * from https://github.com/ef4/rfcs/blob/js-compatible-resolving/text/0000-js-compatible-resolution.md;

* import * from https://github.com/emberjs/rfcs/blob/template-import/text/0000-template-imports.md;

* define this as a preprocessor directive, not an extension to core handlebars

* scroll down and see unresolved questions

## Summary

> One paragraph explanation of the feature.

## Motivation

> Why are we doing this? What use cases does it support? What is the expected
outcome?

## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?

TODO:

* how to use/import things from new world in sloppy templates?
* how to use/import things from addons/node_modules?
* how to avoid importing `../../../../../../..`? `$src`? (does it work in JS too?)
* how to "import" resolver things from strict mode? `$app`? (does it work in JS too?)
  * how does "association" work for old things?
* { `foo-bar/component.js`, `foo-bar/template.hbs` } or { `foo-bar.hbs`, `foo-bar.js` }?
  * if the latter, how does the template import things from the js file? `import { someHelper } from '.'`?
* what is the recommended file/tree layout?
* how do you opt-in to strict mode without any imports? (does it matter?)
* keywords/syntax vs imports list
* something something DI injections
* do we allow JS comments inside `--- ... ---` (to temporarily comment out imports, or docs??)
  * do we allow an optional new line after "closing" ---?
* to answer Robert's question: `--- ... ---` is a "preprocessor directive" and not a handlebars thing (?)
  * implies that hbs` doesn't automatically get imports (which is good/correct/important)
* linting/errors
  * imports and comments only
