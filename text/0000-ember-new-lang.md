- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Ember New Lang

## Summary

Introduce `--lang` flag as an option for `ember new` within Ember-CLI. This will set the `lang` attribute on the html element. A `lang` attribute defines the language of an element or a document.

## Motivation

This will in part address item 4 within issue https://github.com/emberjs/rfcs/issues/595

> Both assistive technologies and conventional user agents can render text more accurately when the language of the Web page is identified.
> -- <cite>[W3C - Understanding Success Criterion 3.1.1: Language of Page](https://www.w3.org/WAI/WCAG21/Understanding/language-of-page.html)</cite>

Having a page language specified should improve the user experience and technical improvements around these areas:

* Screen readers, braille translation software and similar technologies
* Captions with synchronized media (such as video subtitles)
* Correct dictionary lookups for translations
* Assisting search engines
* Improving typography in certain situations
* and more... [Source 1](https://www.w3.org/WAI/WCAG21/Techniques/html/H57) [Source 2](https://www.w3.org/WAI/WCAG21/Understanding/language-of-page.html) [Source 3](https://www.w3.org/TR/1999/REC-html401-19991224/struct/dirlang.html#adef-lang)

## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## How we teach this

The [Ember CLI API documentation](https://ember-cli.com/api/) should be updated to reflect the new flag.

We update the [Ember.js CLI Guides](https://cli.emberjs.com/release/basic-use/cli-commands/) to reflect the new flag much like we demonstrate `--yarn` usage.

The Ember CLI `--help` command should explain what kind of value is expected to be passed to the new flag.

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

* Can the default lang value simply be English? ([Valuable discussion points in this issue](https://cli.emberjs.com/release/basic-use/cli-commands/))

