- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Ember New Lang

## Summary

Introduce `--lang` flag as an option for `ember new` within Ember-CLI. This will set the `lang` attribute on the html element. A `lang` attribute defines the language of an element or a document.

## Motivation

### Why are we doing this?

This will in part address item 4 within issue https://github.com/emberjs/rfcs/issues/595

### What use cases does it support?

Explicitly setting the `lang` attribute on the `html` element. _([W3C - Using the language attribute on the HTML element](https://www.w3.org/WAI/WCAG21/Techniques/html/H57))_

### What is the expected outcome?

> Both assistive technologies and conventional user agents can render text more accurately when the language of the Web page is identified.
> -- <cite>[W3C - Understanding Success Criterion 3.1.1: Language of Page](https://www.w3.org/WAI/WCAG21/Understanding/language-of-page.html)</cite>

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
