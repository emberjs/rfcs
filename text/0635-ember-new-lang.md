---
Start Date: 2020-05-29
Relevant Team(s): CLI, Learning
RFC PR: https://github.com/emberjs/rfcs/pull/635
Tracking: (leave this empty)
Authors: Joseph Sumner, Ava Wroten, Jamie White, Melanie Sumner

---

# Ember New Lang

## Summary

This RFC introduces the `--lang` flag as an option for `ember new`, `ember init`, and `ember addon` commands within the Ember CLI. The feature targets the ember-cli build process -- specifically, when generating the file for the application's entry point at `app/index.html`. If the flag is used with a valid language code, it will assign the `lang` attribute in the file's root `<html>` element to that code. The `lang` attribute is [formally defined within the current HTML5 specification](https://html.spec.whatwg.org/#the-lang-and-xml:lang-attributes); it is used to specify the base human language of an element or a document in a way that can be programmatically understood by assistive technology.

## Motivation

The overall motivation for this RFC is the viewpoint that brand-new Ember apps should not immediately fail [global legal conformance requirements](https://www.lflegal.com/2013/05/gaad-legal/#International-Laws-Regulations-and-Treaties-Impacting-Digital-Accessibility-Partial-Listing) as they pertain to digital accessibility.

The solution presented in this RFC offers the first stage of a resolution to one of the issues documented in the framework's [list of long-standing Technical Accessibility Issues for New Ember Apps](https://github.com/emberjs/rfcs/issues/595) -- specifically, *“Missing default language declaration”* (Section #4). Note: acceptance or implementation of this RFC does not implicitly or explicitly endorse any related RFCs that are related to this issue.

This RFC and its proposed approach have both been developed within the Ember.js Accessibility Strike Team with the explicit objective of helping to ensure that Ember applications achieve [WCAG Success Criterion 3.1.1: Language of Page](https://www.w3.org/WAI/WCAG21/Understanding/language-of-page.html) from the moment they are created. The state of the `lang` attribute has a usability impact on the experience of users that require screen-reading assistive technology. When the attribute is properly assigned:

> "Both assistive technologies and conventional user agents can render text more accurately when the language of the Web page is identified. Screen readers can load the correct pronunciation rules. Visual browsers can display characters and scripts correctly. Media players can show captions correctly. **As a result, users with disabilities will be better able to understand the content.**"
>
> **Source: [WCAG Success Criterion 3.1.1: Intent](https://www.w3.org/WAI/WCAG21/Understanding/language-of-page.html#intent)**

When the language of the page cannot be identified, the integrity of the above information cannot be guaranteed. 
Consider the following use case:

- the application developer is unaware that Ember now includes the lang attribute
- the application does not require internationalization
- the application's content is in a language that is not English
- an end-user with a screen reader turned on, whose operating system (OS) is set to a different language, navigates to that page with their screen reader turned on
- the screen reader would attempt to read the page in the language that is defined by the lang attribute on the page, but the supporting element information ("button", "link", etc) is read out in the language that is set by the operating system.

### Testing it out
To see what happens when this information does not match, we created a new Ember application and created four buttons:

```html
<button type="button">Click Me</button>
<button type="button">点击我</button>
<button type="button">Haz click en mi</button>
<button type="button">Нажми на меня</button>
```

#### No Language Defined

If no lang attribute is set for the page or the parts:

- the screen reader defaults to the operating system (OS) language
- it reads Spanish in an English accent, and the button element was also still read in English 
- for the Chinese and Russian letters, it spelled out the letters (i.e., "Cyrillic Letter E")

#### Language Defined

We then changed the lang attribute value and listened to these buttons in Chinese(zh), Spanish(es) and Russian(ru). Here's what happened:

- in each case, the announcer's voice changed for the content 
- since the OS was set to English, the supporting element information that the assistive tech (AT) announces was in the OS language (English)
- when set to Chinese, the AT read the English, Chinese and Spanish well enough to understand, but did not read out the Russian; likewise, Russian behaved similarly (read all of them except the Chinese)

We then tested what happens if the `lang` attribute was explicitly defined on each of the buttons with the app language set to English:

```html
<button type="button">Click Me</button>
<button type="button" lang="zh">点击我</button>
<button type="button" lang="es">Haz click en mi</button>
<button type="button" lang="ru">Нажми на меня</button>
```

Each were read correctly by AT in their respective languages, followed with the word "button" in the OS language (English).

In this context, it is the users of screen readers, braille translation software, and similar assitive technologies for whom valid page language specifications provide the greatest improvements to user experience and technical operation. It is, however, extremely important to note that that although digital accessibility concerns are the primary motivators for developing formalized page language specifications, achieving WCAG SC-3.1.1 should certainly be considered a global application improvement. The following list contains application use cases that all benefit from having a valid page language specified, but are not strictly tied to digital accessibility requirements or assistive technology:

- Captions with synchronized media (such as video subtitles)
- Correct dictionary lookups for translations
- Assisting search engines
- Improving typography in certain situations

Accordingly, while the primary motivation of this RFC is to address an unresolved digital accessibility issue in Ember, it is expected that a successful implementation of the proposed `--lang` flag solution will provide additional, non-accessibility-related improvements to the baseline quality of new Ember applications.

## Detailed design

Link to [candidate implementation](https://github.com/josephdsumner/ember-cli/compare/master...ember-new-lang-base).

We have explicitly chosen `--lang` as the flag (vs `--language`) for consistency with the HTML attribute itself. 

```bash
ember new my-app --lang en-US
# -l is also a valid alias
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
    alias: -l <value>
```

### Misusage and Error Handling


Broadly, incorrect usage of the `--lang` flag covers three use-case categories:

- The flag has been specified with a programming language as the argument
- The flag has been specified with no argument
- The flag has been specified with an invalid language code

This RFC proposes that these cases cause the build process to halt with a revelant error and help message as opposed to simply reverting to the default value and reporting a message. **The rationale for the recommendation to halt instead of just report is that in these all of these cases, the user has explicitly typed `--lang` into their CLI tool. This is an unambiguous declaration of intent by the user to use the `--lang` flag correctly.**



#### Invalid Language Codes

Language codes are verified against [is-language-code](https://www.npmjs.com/package/is-language-code). (see [examples of valid ISO country codes](https://en.wikipedia.org/wiki/List_of_ISO_3166_country_codes))

If an invalid language code is given such as `--lang en-UK` the indended output should be a shell error that will halt the build.

```bash
ember new my-app --lang en-UK
Unrecognized language subtag, "uk".
```

#### No argument passed
Additionally, if the `--lang` flag is used but no argument is defined, e.g., `ember new my-app --lang`, the build will also be halted. An error message and usage info should be shown in these cases:

```bash
# Detect trailing option (declared)
ember new my-app --lang --skip-git

An error with the `--lang` flag returned the following message:
  Detected lang specification starting with command flag `-`.
  Is `--skip-git` meant to be an ember-cli command option?
  This issue is likely caused by using the `--lang` flag without a specification.
Information about using the `--lang` flag:
  The `--lang` flag sets the base human language of the app in index.html
  If used, the lang option must specfify a valid language code.
  For default behavior, remove the flag.
  See `ember <command> help` for more information.
```

```bash
# Detect trailing option (undeclared)
ember new my-app --lang

An error with the `--lang` flag returned the following message:
  Detected lang specification starting with command flag `-`.
  Is `--disable-analytics` meant to be an ember-cli command option?
  This issue is likely caused by using the `--lang` flag without a specification.
Information about using the `--lang` flag:
  The `--lang` flag sets the base human language of the app in index.html
  If used, the lang option must specfify a valid language code.
  For default behavior, remove the flag.
  See `ember <command> help` for more information.
```

#### Common Misunderstandings: Programming Languages

A developer may encounter the flag and make incorrect assumptions about what it can mean. Such as, `-l typescript` or `-l glimmer`. Such incorrect assumptions will be manually caught by the implementation and the developer will be shown a friendly error message such as the following:


```bash
ember new my-app --lang=typescript

An error with the `--lang` flag returned the following message:
  Trying to set the app programming language to `typescript?`
  This is not the intended usage of the `--lang` flag.
Information about using the `--lang` flag:
  The `--lang` flag sets the base human language of the app in index.html
  If used, the lang option must specfify a valid language code.
  For default behavior, remove the flag.
  See `ember <command> help` for more information.
```

## How we teach this

1. Update the [Ember CLI API documentation](https://ember-cli.com/api/) to reflect the new flag.
2. Update the [Ember.js CLI Guides](https://cli.emberjs.com/release/basic-use/cli-commands/) to reflect the new flag much like we demonstrate `--yarn` usage.
3. Update the Ember CLI `--help` command so it explains what kind of value is expected to be passed to the `--lang` flag.
4. Update the Super Rental tutorial to include updated information.
5. Update the Guides - specifically the section that discusses the language attribute. 

### For the Ember CLI Guides:

The `--lang` flag can be used to set the spoken language of the app or addon.

To use with a language code only, this is the syntax that would be used: 

```bash
ember new my-app --lang en
```

To use with a language code and a region code:

```bash
ember new my-app --lang en-US
```

An error will be thrown if the country and region codes are incorrect. Additionally, helpful error text has been added in cases where the developer misunderstands the `--lang` flag and thinks that it is the programming language rather than the HTML attribute.

### For the Ember.js Guides

Specifically, update to https://guides.emberjs.com/release/accessibility/application-considerations/#toc_language-attribute:

Every application must have a primary language declaration. This language declaration is important for assistive technology like screen readers, internationalization tools built into browsers, and search engines.

To indicate the primary language, use the `--lang` flag when generating a new Ember app. This is inherited by all other elements, and will set a default language for the text in the document head element. If app globalization is desired, then consider using the `ember-intl` addon.

If there happens to be any content on the page that is in a different language from that declared in the <html> element, the `lang` attribute can be used on the parent element to indicate a different language.

Note: While an app cannot have multiple language attribute values defined at the same time, the language of specific elements can be defined to be different than the language of an app. For example, a language (e.g., `lang="en"`) could be set on the page's HTML element and then a different language (e.g., `lang="es"`) could be set on a different element in the page content (if appropriate).

### As this relates to ember-intl

The popular [localization library ember-intl](https://github.com/ember-intl/ember-intl) does not conflict with the addition of this new Ember CLI flag. The addition of this flag offers some out of the box support where it was previously missing for new Ember apps. It is still recommended that globalized apps leverage `ember-intl`.

## Drawbacks

* More flags means more combinations of ways to run `ember new` which can be hard to test for and is potentially unsustainable.
* Users may be confused about whether or not they are supposed to specify a human language or a programming language (i.e. `--lang typescript`). However, we think we've mitigated this by using the HTML attribute as the flag name, and having helpful error messages that guide developers in the right direction.

## Alternatives
These are the alternative approaches that we are aware of; if more become apparent in discussion, this RFC will be updated to include them. 

- Set the default html lang attribute to `en-US` (the language of the Ember.js project) and assume users will either change the `lang` value themselves, or rely exclusively on `ember-intl` (and not just for apps that require full globalization).
  - [Valuable discussion points in this issue](https://github.com/emberjs/rfcs/issues/595)
  > The data we already have suggests that most Ember applications are:... in English... use internationalization if other languages are required
  - We have existing art in other frameworks (Vue sets `lang="en"` by default)
  - It's consistent with the "80%" rule (solve for 80% of the use cases)
  - We prop up this new default with supporting Ember documentation to describe to users how to use ember-intl to choose another language along with the potential "bug" of an Ember app being interpreted as the wrong language.
- Make no flag change
  - By having no `lang` attribute an Ember app will default to using the system OS language.
  - Update the Ember documentation to include the pro's and con's of setting a language, along with how to do so.

## Unresolved questions

No unresolved questions currently but if RFC discussion yields additional unresolved questions, we will add them here.

## References

- [Understanding Success Criterion 3.1.1: Language of Page](https://www.w3.org/WAI/WCAG21/Understanding/language-of-page.html)
- [Technique H57: Using the language attribute on the HTML element](https://www.w3.org/WAI/WCAG21/Techniques/html/H57)
- [HTML5 Specification: `lang` attribute](https://html.spec.whatwg.org/#the-lang-and-xml:lang-attributes)
