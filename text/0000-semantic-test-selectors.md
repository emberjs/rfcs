- Start Date: 2018-04-06
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Semantic Test Selectors

## Summary

Use perceivable text to target elements in UI tests.

**Before:**

```js
await fillIn('.login-form .field-email', 'alice@example.com');
await fillIn('.login-form .field-password', 'topsecret');
await click('.submit-btn');
```

**After:**

```js
await fillIn('Email', 'alice@example.com');
await fillIn('Password', 'topsecret');
await click('Log in');
```

## Motivation

This idea was first pitched in the talk [Say More] at EmberConf 2018.

There are 3 overarching motivations for this change. In order of importance:

### Access via accessibility

We, as a community, want Ember apps to be accessible by default. We want to
provide first-class experiences to users of assistive technologies. We want
these values to be baked into our tools.

We can use [axe-core] via [ember-a11y-testing] to explicitly audit our UIs for
accessibility. While this is extremely valuable, it involves active
participation on the part of the developer. What if, instead, our UI tests
implicitly asserted that our UIs meet the community’s standards for
accessibility. Therefore, the act of writing a UI test would also guarantee
that the key flows in an app are accessible.

One step we can take is to impose “access via accessibility”. That is to say,
we can interact with our UI the same way our users do: by perceivable semantic
labels. If an element is not meaningfully labelled, it is likely invisible to
users of assistive technologies and therefore should be invisible to our tests
too. The corollary: if we want to access a UI element in a test, we do so as if
we were using a screen reader.

### The rule of least power

> Expressing constraints, relationships and processing instructions in less
> powerful languages increases the flexibility with which information can be
> reused: the less powerful the language, the more you can do with the data
> stored in that language.
>
> — <cite>Tim Berners-Lee & Noah Mendelsohn, [The Rule of Least Power]</cite>

Currently, CSS selectors allow us to target elements with arbitrary
specificity. The more we over-specify in our tests, the less room we leave
ourselves to reinterpret those tests in the future.

The language of UI tests is already wonderfully constrained: `visit`, `click`,
`fillIn`, `find`. These helpers, simple at the surface, encode an enormous
amount of meaning — meaning that is shared between all Ember developers and
evolves over time. If we could similarly contstrain the means by which we
select elements, we can open the door for further evolution.

### Cleaner tests

This is perhaps the most visceral motivation. Which test would you rather read?

This one:

```js
await fillIn('.login-form .field-email', 'alice@example.com');
await fillIn('.login-form .field-password', 'topsecret');
await click('.submit-btn');
```

Or this one:

```js
await fillIn('Email', 'alice@example.com');
await fillIn('Password', 'topsecret');
await click('Log in');
```

## Detailed design

We will provide semantic variants of `click` and `fillIn`. Their proposed
behaviours are given below.

### Semantic `click`

```ts
function click(label: string): Promise<void>
```

This helper finds a **link** or **button** whose **perceivable text** contains
the string specified by `label`, or a **switch** whose **perceivable label**
contains the string specified by `label`.

If it finds exactly one matching element, it invokes the general-purpose
`click` helper upon it.

If it finds multiple matching elements, it throws:

```
Found multiple clickable elements with perceivable text '${label}' when calling
`click('${label}')`. You may want to use a more descriptive label in the test
and the template.
```

If it finds no matching element, it throws:

```
Clickable element with perceivable text '${label}' not found when calling `click('${label}')`.
```

### Semantic `fillIn`

```ts
function fillIn(label: string, value: string): Promise<void>
```

This helper finds an **input** whose **perceivable label** contains the string
specified by `label`.

If it finds a matching element, it invokes the general-purpose `fillIn` helper
upon it with `value` as the second argument.

If a label is found without a counterpart form control it throws:

```
Form control not found for label containing '${label}' when calling
`fillIn('${label}', '${value}')`. You may want to check the `for` attribute of
the label.
```

If no label is found, it throws:

```
Form control labelled '${label}' not found when calling `fillIn('${label}', '${value}')`.
If the form control exists, you may want to label its using a label, title or aria-label.
```

### Definitions

A **link** is one of the following:

- `a`
- `[role="link"]`
- `[role="menuitem"]`

A **button** is one of the following:

- `button`
- `input[type="button"]`
- `input[type="reset"]`
- `input[type="submit"]`
- `[role="button"]`

An **input** is one of the following:

- `input`
- `textarea`
- `select`
- `[role="slider"]`
- `[role="spinbutton"]`
- `[role="textbox"]`
- `[contenteditable="true"]`

A **switch** is one of the following:

- `input[type="checkbox"]`
- `input[type="radio"]`
- `[role="checkbox"]`
- `[role="option"]`
- `[role="radio"]`

A **form control** is an **input** or **switch**.

The **perceivable text** of a **link**, **button**, or other element may be
found in any/all of the following locations **if-and-only-if the element is
perceivable**:

The percivable text is calculated using [Text alternative spec](https://www.w3.org/TR/accname-1.1/#mapping_additional_nd_te)

**tldr**
A. if current node is hidden
 return ""
B. if current node has `aria-labeledby` accumulate values and
  return accumulated string
C. if current node has `aria-label`  && not nested within a `label`
  return value
D. if node has attribute or referenced by a text alternative option
  return value
E. if it is nested within a `label`
  return name based on section E
F. try generate if role allows [try name from content](https://www.w3.org/TR/wai-aria-1.1/#namefromcontent)
              || current node is referenced by aria-labelledby, aria-describedby
              || or is a descendant of a native host language text alternative element
    accumulates name from
      - `:before`, `:after` text alternative
      - the result of text alternative for each child node

For an element to be **perceivable**, it MUST be **perceivable to all users**.
If an element is not perceivable to screen readers, or not perceivable due to
unsufficient colour contrast, then it is not perceivable to tests either.

Perceivability takes into account the DOM, CSS and the accessbility tree. If an
element or one of its ancestors has `display: none`, `visiblity: hidden`,
`hidden="true"` or `aria-hidden="true"`, then it is **not perceivable**. This
is not a complete definition of perceivability in the browser, but a reasonable
approximation for the purposes of this document.

### Distribution

The helpers described above will be distributed in a standalone package
(e.g. `ember-semantic-test-helpers`). They may be imported into a tests as follows:

```js
import { click, fillIn } from 'ember-semantic-test-helpers';
```

If it is desirable to use both the semantic and selector based helpers
side-by-side, this can be achieved as follows:

```js
import { click, fillIn } from 'ember-semantic-test-helpers';
import { click as clickBySelector, fillIn as fillInBySelector } from '@ember/test-helpers';
```

### Underlying Implementation

Kent C. Dodds’ [dom-testing-library] appears to be well-designed, framework
agnostic, and well worth evaluating as the underlying engine for Ember’s
semantic test selectors. The implementations of [`queryByText`] and
[`queryByLabelText`] display a considerable amount of rigour and seem
sufficiently composable for our needs.

### Candidate Implementation

[@billybonks] has created an implementation of `ember-semantic-test-helpers`
that we can all try out today:

> So i have been doing this in our app since ember-conf, extracted it into an
> addon, that people can try play with.
>
> Some differences exist instead of just fillin i have fillin select toggle, an
> example of how fillin differs to select is with power-select if you use select
> helper it will use selectChoose if you use select helper it will first use
> selectSearch then selectChoose.
>
> It currently doesn't fully respect the spec with regards to what a button is or
> a checkbox.
>
> https://github.com/tradegecko/ember-semantic-test-helpers
>
> I will be working over the next few days, to make sure it respects those
> definitions and has proper tests/documentation.
>
> — <cite>[@billybonks], [comment #383054032]</cite>

## How we teach this

This proposal is an evolution of current Ember UI testing patterns. We believe
that semantic testing should eventually become the default approach, and that
the guides should reflect that. We propose also that the guides reflect the
motivation for semantic testing, as laid out in this RFC.

### More Prescriptive, More Opinionated

It is tempting to claim that this style of testing is “simpler” but in reality
the cognitive overhead we remove from writing tests will reappear elsewhere.

We are deliberately aiming to make Ember UI testing more prescriptive and
convention-driven. Where previously the developer needed to consider how best
to target a particular link, button, or form control, now we are presenting one
recommended approach and the developer must instead make their templates match
expectations.

This reduces the number of **decisions** required when writing tests and
templates, but increases the number of **rules** to learn when writing
templates. Fortunately, the rules in question are web standards and WCAG
guidelines—if you care about building accessible apps, you’ll want to learn
them anyway.

### First Class Errors

Errors will play a big part in how we teach this. When one of the helpers fails
to find a matching element, the error message can go *beyond informative* and
actively recommend changes the developer may want to make, linking to
documentation and examples. We should shoot for errors of the quality the
[Elm compiler] or [axe-core] produce.

### Escape Valves

If particular parts of an app’s UI are too difficult to target using semantic
helpers, the developer can fall back to the selector-based helpers:

```js
import { module, test } from 'qunit';
import { setupApplicationTest } from 'ember-qunit';
import { click, fillIn } from 'ember-semantic-test-helpers';
import { click as clickBySelector } from '@ember/test-helpers';

module('Authentication', function(hooks) {
  setupApplicationTest(hooks);

  test('Logging in', async function(assert) {
    await visit('/login');
    await fillIn('Email', 'alice@example.com');
    await fillIn('Password', 'topsecret');
    await clickBySelector('.my-complex-component');
    await click('Log in');

    assert.dom('main').hasText('Logged in');
  });
});
```

### Road to Adoption

We’ll use the following two shorthands throughout this section:

- **2018 style** is the style of testing proposed in [RFC #268] and shipped in
  early 2018 (explicit helper imports and explicit async/await)
- **Classic style** is the former testing style (global helpers and implicit
  async/await)

#### Stage 1: Early Adoption

In this stage the aim is to make it easy for developers to adopt semantic test
helpers incrementally, one test module at a time, using the following process:

1. If the project’s UI tests are written in **classic style**, first upgrade
   them to **2018 style** using [ember-test-helpers-codemod].

2. Install the semantic test helpers package:

   ```sh
   $ ember install ember-semantic-test-helpers
   ```

3. For each UI test module:

   1. Import helpers from the semantic module:

      ```diff
      - import { visit, find, click, fillIn } from '@ember/test-helpers';
      + import { visit, find } from '@ember/test-helpers';
      + import { click, fillIn } from 'ember-semantic-test-helpers';
      ```

   2. Update invocations to use semantic labels:

      ```diff
        test('logging in', async function() {
      -   await fillIn('.login-form .field-email', 'alice@example.com');
      +   await fillIn('Email', 'alice@example.com');
      -   await fillIn('.login-form .field-password', 'topsecret');
      +   await fillIn('Password', 'topsecret');
      -   await click('.submit-btn');
      +   await click('Log in');

          assert.dom('main').hasText('Logged in');
        });
      ```

   3. Fix errors reported by the semantic helpers:

      ```diff
      - <input type="email" value={{email}}>
      + <label for="field-email">Email</label>
      + <input id="field-email" type="email" value={{email}}>
      ```

#### Stage 2: Codemod

We’ll provide a codemod to automate the process described above.

When transforming test helper invocations, the codemod will attempt to find an
appropriate semantic label. If no such label can be found, the codemod will
report a transformation failure, prompting the developer to fix the DOM such
that an unambiguous semantic label is present.

Exact codemod design TBC.

#### Stage 3: Make it the Default

The goal of this stage is to make semantic test helpers the default for Ember
projects. To achieve this, we:

- Include `ember-semantic-test-selectors` as a dependency in the `app` blueprint
- Modify test blueprints to `import { click, fillIn } from 'ember-semantic-test-selectors'`

With this approach, existing tests are unaffected but new tests will be
generated to use the semantic style.

## Drawbacks

### Brittle Tests

This approach risks making tests brittle in the face of copy changes, likely
exacerbated in internationalized apps where copy changes can be me made far
from the templates they affect.

This risk should be given serious consideration when evaluating this RFC.

### Complex Components

While this approach can be adopted incrementally (and to opt-out the developer
need only import the general purpose helpers) there are likely a great deal of
Ember components out there that will be awkward to target in this manner.

It may be worth running a survey of addons and apps to identify the most complex
custom input components, such that we can devise recommendations for
integrating them with semantic test selectors.

### Non-unique Labels

What if you have multiple “Save” buttons on screen at once, or multiple inputs
with the same label? Semantic test helpers can’t disambiguate. In these
situations we ask the developer to consider: *should these buttons or inputs
actually have unique labels?*

Even if the visual design requires repeated labels, the developer can use the
`title` or `aria-label` attributes to provide more specific labels that can be
targeted in tests and also provide more context to users of assistive
technologies.

For example, given the form:

```hbs
<form>
  <h4>Receive notifications via:</h4>

  <ul>
    <li>
      <input type="checkbox" id="notifications-email">
      <label for="notifications-email">Email</label>
    </li>
    <li>
      <input type="checkbox" id="notifications-web">
      <label for="notifications-web">Web</label>
    </li>
  </ul>

  <h4>Receive requests via:</h4>

  <ul>
    <li>
      <input type="checkbox" id="requests-email">
      <label for="requests-email">Email</label>
    </li>
    <li>
      <input type="checkbox" id="requests-web">
      <label for="requests-web">Web</label>
    </li>
  </ul>
</form>
```

We can add `title` attributes to our labels:

```diff
- <label for="notifications-email">Email</label>
+ <label for="notifications-email" title="Receive notifications via Email">Email</label>
```

And subsequently target them in our tests with:

```js
await click('Receive notifications via Email');
```

## Alternatives

### Do Nothing

We could, of course, continue targetting elements in tests with CSS selectors.
However, we hope we’ve provided a sufficiently persuasive argument in the
“Motivation” section for why this presents an overall improvement for Ember UI
Testing.

### Compose Primitive Helpers

We could instead take the [react-testing-library] approach and recommend
developers use `getByText` and `getByLabelText` helpers in conjunction the
general purpose `click` and `fillIn`.

```js
await fillIn(getByLabelText('Email'), 'alice@example.com');
await fillIn(getByLabelText('Password'), 'alice@example.com');
await click(getByText('Login'));
```

This would increases the amount of line noise in tests but is more explicit and
less “magical”.

[@cibernox] proposes a similar tactic in [comment #28667926]:

```js
import { click, fillIn, triggerKeyEvent, findInput } from '@ember/test-helpers';

let input = findInput('Label');
await fillIn(input, 'Search term');
await trigerKeyEvent(input, 'keydown', 'Enter');
```

### Distribute as Part of @ember/test-helpers

We could distribute the semantic helpers as part of [@ember/test-helpers].

With this approach, imports might look like this:

```js
import { click, fillIn } from '@ember/test-helpers/semantic';
```

Looking much further down the road, the semantic helpers could replace the
top-level `click` and `fillIn` exports, thereby forcefully becoming the defaults.

This would import semantic helpers:

```js
import { visit, find, click, fillIn } from '@ember/test-helpers';
```

To fall back to selector-based helpers:

```js
import {
  visit,
  find,
  clickBySelector as click,
  fillInBySelector as fillIn
} from '@ember/test-helpers';
```

### Overload the existing `click` and `fillIn` helpers

[@cibernox] proposed the following:

> What I don't like a lot is having two fillIn or click methods, and I don't
> think we can only rely on semantic selectors because there there will be
> exceptions or developers that just won't care about semantics.
>
> I am thinking that perhaps after some experimentation, we could unify them.

```js
// click.js in @ember/test-helpers

export default async function(selectorOrText) {
  if (selectorOrText instanceof HTMLElement || isSelector(selectorOrText)){
    // current behaviour
  } else {
    // semantic behaviour
  }
}
```

> That way we wouldn't have to choose between semantic vs css-based when importing the helpers.
>
> — <cite>[@cibernox], [comment #382852271]</cite>

After some discussion, we posited that implementing `isSelector` reliably could be
devilishly difficult.

> touché, and with custom element the task is essentially impossible. I take it
> back then.
>
> — [@cibernox], [comment #382859189]

Then [@cibernox] came upon with some ingenious alternatives:

> Perhaps the very test selectors could be expanded to take a POJO as first
> argument an allow more invocations:

```js
fillIn(input, 'text'); // Receiving an HTMLElement
fillIn('.css-class', text); // Css selector
fillIn({ label: 'Email' }, 'text'); // Semantic descriptor POJO
```

> Perhaps even if we can implement a `isSelector` that is 99% accurate, we could
> rely on it and for the 1% of cases in which the text of the link looks like a
> css selector but it's not, we can force it with

```js
fillIn(input, 'text'); // Receiving an HTMLElement
fillIn('.css-class', text); // Css selector
fillIn('Email', 'text'); // Clearly not a selector, interpreted as label
fillIn({ label: '#holidays' }, 'text'); // Looks like a selector but it's the actual text of the link to a hashtag
fillIn({ selector: 'my-foo' }, 'text'); // The other way aroind, it may seem a label but it's actually a custom element.
```

### Prior Art

#### Capybara

[Capybara] is a high level Ruby library for scripting UI interactions. It is
often used for acceptance testing Rails apps in a manner that should feel very
familiar to Emberistas. [Capybara’s testing helpers] behave in a similar manner
to the helpers proposed in this RFC:

```rb
fill_in('Email', with: 'Alice')
fill_in('Password', with: 'topsecret')
click_on('Log in')
```

The implementation of [`fill_in`], [`click_on`], and the underlying
[`fillable_field`], [`locate_field`], and [`link_or_button`] are all
instructive but closely tied to Capybara’s internal infrastructure and
therefore probably not sufficiently portable for our purposes.

#### React-testing-library

The recently-announced [react-testing-library] is a lightweight, opinonated
approach to React testing. It’s lower-level than Capybara or Ember Testing, but
shares some key principles with this RFC, particularly in regard to
accessibility.

> This library encourages your applications to be more accessible and allows
> you to get your tests closer to using your components the way a user will,
> which allows your tests to give you more confidence that your application
> will work when a real user uses it.
>
> — <cite>Kent C. Dodds, [Introducing the react-testing-library]</cite>

In practice it looks something like this:

```js
let { getByLabelText, getByText } = render(<LoginForm />);

getByLabelText('Email').value = 'alice@example.com';
getByLabelText('Password').value = 'topsecret';
Simulate.click(getByText('Log in'));
```

As noted above, [dom-testing-library] (the library that powers
react-testing-library) may provide a good basis for Ember’s semantic test
helpers.

## Unresolved questions

### Switches

This proposal demonstrates interacting with **switches** via `click` rather
than `fillIn`. It may be desirable to make both work in a [“do what I mean”]
manner.

```js
// Toggles the current state of the checkbox
await click('Receive notifications via Email');

// Sets the checkbox to checked
await fillIn('Receive notifications via Email', true);
```

### Semantic Assertions

This RFC has nothing to say about UI test assertions and how we bring the
semantic approach to statements like this:

```js
assert.dom('[data-test-flash]').hasText('Some message');
```

Discussion of this aspect of testing is left to the comment thread and future
RFCs.


[Say More]: https://youtu.be/qfnkDyHVJzs?t=5h39m15s
[axe-core]: https://axe-core.org/
[ember-a11y-testing]: https://github.com/ember-a11y/ember-a11y-testing
[The Rule of Least Power]: https://www.w3.org/2001/tag/doc/leastPower
[RFC #268]: https://github.com/emberjs/rfcs/blob/master/text/0268-acceptance-testing-refactor.md
[ember-test-helpers-codemod]: https://github.com/simonihmig/ember-test-helpers-codemod
[dom-testing-library]: https://github.com/kentcdodds/dom-testing-library
[react-testing-library]: https://github.com/kentcdodds/react-testing-library
[`queryByText`]: https://github.com/kentcdodds/dom-testing-library/blob/131a20bcc7cca4c0853acc49365a6c4746165471/src/queries.js#L47-L53
[`queryByLabelText`]: https://github.com/kentcdodds/dom-testing-library/blob/2273f03300dbdb360e02e08f73fd053fe2e82c80/src/queries.js#L19-L45
[Elm compiler]: http://elm-lang.org/blog/compiler-errors-for-humans
[axe-core]: https://axe-core.org/docs/
[Capybara]: http://teamcapybara.github.io/capybara/
[Capybara’s testing helpers]: http://www.rubydoc.info/github/teamcapybara/capybara/master#Clicking_links_and_buttons
[`fill_in`]: https://github.com/teamcapybara/capybara/blob/9b3c4fb4115e0c3890f4203b36c913706c0a574c/lib/capybara/node/actions.rb#L83-L86
[`click_on`]: https://github.com/teamcapybara/capybara/blob/9b3c4fb4115e0c3890f4203b36c913706c0a574c/lib/capybara/node/actions.rb#L23-L26
[`fillable_field`]: https://github.com/teamcapybara/capybara/blob/9b3c4fb4115e0c3890f4203b36c913706c0a574c/lib/capybara/selector.rb#L239-L282
[`locate_field`]: https://github.com/teamcapybara/capybara/blob/383211de9f6176a66bf847e1fded0728f9c82bfd/lib/capybara/selector/selector.rb#L235-L251
[`link_or_button`]: https://github.com/teamcapybara/capybara/blob/9b3c4fb4115e0c3890f4203b36c913706c0a574c/lib/capybara/selector.rb#L224-L237
[Introducing the react-testing-library]: https://blog.kentcdodds.com/introducing-the-react-testing-library-e3a274307e65
[“do what I mean”]: https://en.wikipedia.org/wiki/DWIM
[@ember/test-helpers]: https://github.com/emberjs/ember-test-helpers
[@cibernox]: https://github.com/cibernox
[comment #382852271]: https://github.com/emberjs/rfcs/pull/327#issuecomment-382852271
[comment #382859189]: https://github.com/emberjs/rfcs/pull/327#issuecomment-382859189
[comment #28667926]: https://github.com/emberjs/rfcs/pull/327#commitcomment-28667926
[@billybonks]: https://github.com/billybonks
[comment #383054032]: https://github.com/emberjs/rfcs/pull/327#issuecomment-383054032
