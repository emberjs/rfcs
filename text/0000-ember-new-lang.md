- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Ember New Lang

## Summary

This RFC introduces the `--lang` flag as an option for `ember new` and `ember init` commands within the Ember CLI. The feature targets the ember-cli build process -- specifically, when generating the file for the application's entry point at `app/index.html`. If the flag is used with a valid language code, it will assign the `lang` attribute in the file's root `<html>` element to that code. The `lang` attribute is [formally defined within the current HTML5 specification](https://html.spec.whatwg.org/#the-lang-and-xml:lang-attributes); it is used to specify the base human language of an element or a document in a way that can be programmatically understood by assistive technology.

## Motivation

The overall motivation for this RFC is the viewpoint that brand-new Ember apps should not immediately fail legal conformance requirements as they pertain to digital accessibility.

The solution presented in this RFC offers the first stage of a resolution to one of the issues documented in the framework's [long-standing list of Technical Accessibility Issues for New Ember Apps](https://github.com/emberjs/rfcs/issues/595) -- specifically, *“Missing default language declaration”* (Section #4).

This RFC and its proposed approach have both been developed within the Ember.js Accessibility Strike Team with the explicit objective of helping to ensure that Ember applications achieve [WCAG Success Criterion 3.1.1: Language of Page](https://www.w3.org/WAI/WCAG21/Understanding/language-of-page.html) from the moment they are created. The state of the `lang` attribute has a usability impact on the experience of users that require screen-reading assistive technology. When the attribute is properly assigned:

> "Both assistive technologies and conventional user agents can render text more accurately when the language of the Web page is identified. Screen readers can load the correct pronunciation rules. Visual browsers can display characters and scripts correctly. Media players can show captions correctly. **As a result, users with disabilities will be better able to understand the content.**"
>
> **Source: [WCAG Success Criterion 3.1.1: Intent](https://www.w3.org/WAI/WCAG21/Understanding/language-of-page.html#intent)**

When the language of the page cannot be identified, the integrity of the above information cannot be guaranteed. In this context, it is the users of screen readers, braille translation software, and similar assitive technologies for whom valid page language specifications provide the greatest improvements to user experience and technical operation. It is, however, extremely important to note that that although digital accessibility concerns are the primary motivators for developing formalized page language specifications, achieving WCAG SC-3.1.1 should certainly be considered a global application improvement. The following list contains application use cases that all benefit from having a valid page language specified, but are not strictly tied to digital accessibility requirements or assistive technology:

- Captions with synchronized media (such as video subtitles)
- Correct dictionary lookups for translations
- Assisting search engines
- Improving typography in certain situations

Accordingly, while the primary motivation of this RFC is to address an unresolved digital accessibility issue in Ember, it is expected that a successful implementation of the proposed `--lang` flag solution will provide additional, non-accessibility-related improvements to the baseline quality of new Ember applications.

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

* Set the default html lang attribute to `en-US` and assume users will use ember-intl if they choose another language.
  - [Valuable discussion points in this issue](https://github.com/emberjs/rfcs/issues/595)
  > The data we already have gives us evidence that most Ember applications are:... in English... use internationalization if other languages are required
  - We prop up this new default with supporting Ember documentation to describe to users how to use ember-intl to choose another language along with the potential "bug" of an Ember app being interpreted as the wrong language.
* Do nothing
  - By having no `lang` attribute an Ember app will default to using the system OS language.
  - By doing nothing, we may wish to at least update the Ember documentation to include the pro's and con's of setting a language along with how to do so.

## Unresolved questions

* Can the default lang value simply be English?
  - We're going with "no" here in the RFC prose. Evidence for our "no" is covered in the Motivation section above. We believe that offering the user a chance to intentionally choose a language outweighs the cons of adding an additional step to simply hit the "80% rule".
  - See above under the Alternatives heading for arguments for "yes".
* Should the attribute be `--language` instead of `--lang`?
  - We're going with "no" here in the RFC prose. Using `lang` more closely connects this flag with the HTML attribute `lang` and distances itself from the potential drawback, mentioned above, of a user thinking this specifies a programming language.

## References

- [Understanding Success Criterion 3.1.1: Language of Page](https://www.w3.org/WAI/WCAG21/Understanding/language-of-page.html)
- [Technique H57: Using the language attribute on the HTML element](https://www.w3.org/WAI/WCAG21/Techniques/html/H57)
- [HTML5 Specification: `lang` attribute](https://html.spec.whatwg.org/#the-lang-and-xml:lang-attributes)
