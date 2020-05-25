- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Ember New Lang

## Summary

Introduce `--lang` flag as an option for `ember new` within Ember-CLI. This will set the `lang` attribute on the html element. A `lang` attribute defines the language of an element or a document.

## Motivation

This RFC offers a partial resolution to technical accessibility issue 4 “Missing default language declaration” from the Ember A11y Strike Team’s [“Technical Accessibility Issues for New Ember Apps” list](https://github.com/emberjs/rfcs/issues/595). This RFC explicitly does _not_ aim to add a default language, but instead aims to add tooling that offers users a way to specify a language.

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

Link to [candidate implementation](https://github.com/josephdsumner/ember-cli/compare/master...ember-new-lang-base).

```bash
ember new my-app --lang en-US
# -lang or -l are valid aliases as well
```

The above ember-cli command will result in the following `index.html` header change.

```html
<html lang="en-US">
```

The flag is added to relevant ember-cli help commands, such as the following:

```bash
ember help new
ember new <app-name> <options...>
  ...
  --lang (String) (Default: "") Sets the base human language of the application via index.html
    aliases: -l <value>, -lang <value>
```

### Invalid Language Codes

Language codes are verified against [is-language-code](https://www.npmjs.com/package/is-language-code). (see [examples of valid ISO country codes](https://en.wikipedia.org/wiki/List_of_ISO_3166_country_codes))

If an invalid language code is given such as `--lang en-UK` the indended output should be a shell error that will hault the build.

```bash
ember new my-app --lang en-UK
Unrecognised language subtag, "uk".
```

#### Common Misunderstandings

A developer may encounter the flag and make incorrect assumptions about what it can mean. Such as, `-l typescript` or `-l glimmer`. Such incorrect assumptions will be manually caught by the implementation and the developer will be shown a friendly error message such as the following:

```bash
ember new my-app --lang typescript
Trying to set the app programming language to typescript? The `--lang flag sets the base human language of the app in index.html
```

## How we teach this

1. Update the [Ember CLI API documentation](https://ember-cli.com/api/) to reflect the new flag.
2. Update the [Ember.js CLI Guides](https://cli.emberjs.com/release/basic-use/cli-commands/) to reflect the new flag much like we demonstrate `--yarn` usage.
3. Update the Ember CLI `--help` command so it explains what kind of value is expected to be passed to the `--lang` flag.

## Drawbacks

* More flags means more combinations of ways to run `ember new` which can be hard to test for and is potentially unsustainable.
* Users may be confused about whether they’re supposed to specify a human language or a programming language (i.e. `--lang typescript`).

## Alternatives

Do nothing.

## Unresolved questions

* Can the default lang value simply be English? ([Valuable discussion points in this issue](https://cli.emberjs.com/release/basic-use/cli-commands/))
* Should the attribute be `--language` instead of `--lang`?
  - We're going with "no". Using `lang` more closely connects this flag with the HTML attribute `lang` and distances itself from the potential drawback, mentioned above, of a user thinking this specifies a programming language.
