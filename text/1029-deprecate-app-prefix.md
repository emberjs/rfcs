---
stage: accepted
start-date: 2024-05-20T00:00:00.000Z
release-date:
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - learning
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1029
project-link:
---

<!---
Directions for above:

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
-->

# Deprecate app-prefix, app-suffix, tests-prefix, and tests-suffix

## Summary

ember-cli addons can use their `contentFor` method to emit arbitrary Javascript into many places. This RFC proposes deprecating and removing several of them:

 - app-prefix
 - app-suffix
 - tests-prefix
 - tests-suffix
 - vendor-prefix
 - vendor-suffix

## Motivation

All of these assume there's going to be one "app" bundle, one "vendor" bundle, and one "tests" bundle. But those assumptions are now nonsense, given code-splitting and builds that can directly evaluate the module graph in the browser.

They are also seldom used based on code searches on emberobserver.com:

| Feature  | Usage Count   | Relevance |
| ---------| ------------- | -----------------------------------------------------------------------------------   |
| app-prefix  | 2 addons   | Last updated 6 and 8 years ago                                     |
| app-suffix | 1 addon     | Last updated 6 years ago |
| tests-prefix | 0 addons  | Never appears in emberobserver at all |
| tests-suffix | 0 addons  | Only some embroider testing infrastructure ever refers to this at all. |
| vendor-prefix | 3 addons | Last updated 6, 7, and 9 years ago | 
| vendor-suffix | 2 addons | Last updated 6 and 9 years ago |

## Transition Path

For all of these types of `contentFor` we will emit a deprecation if an addon returns content for the given `type` argument. An example of the deprecated behavior looks like:

```js
   contentFor(type, config, contents) {
     if (type === 'app-prefix') {
       return `console.log("LOL");`
     }
    }
```

## How We Teach This

The CLI docs [mention contentFor](https://ember-cli.com/api/classes/addon#method_contentFor) but they don't actually document any of the use cases we are deprecating. They describe using contentFor to target index.html instead. That is not being deprecated by this RFC.


### Deprecation Guide Content

#### app-prefix

Returning content from an addon's `contentFor()` hook for `type="app-prefix"` is deprecated. Addons will no longer be allowed to inject arbitrary javascript here. If you need to provide code that apps will run before booting, document that app authors should import and call your code at the start of their own `app.js` file.

#### app-suffix

Returning content from an addon's `contentFor()` hook for `type="app-suffix"` is deprecated. Addons will no longer be allowed to inject arbitrary javascript here. If you need to provide code that apps will run before booting, document that app authors should import and call your code at the start of their own `app.js` file.

If you were using app-suffix to overwrites modules provided by the app, that is intentionally not supported. Adjust your API to tell app authors to import your code and invoke it where appropriate.

#### tests-prefix

Returning content from an addon's `contentFor()` hook for `type="tests-prefix"` is deprecated. Addons will no longer be allowed to inject arbitrary javascript here. Provide utilities that users can import into their own test setup code instead.

#### tests-suffix

Returning content from an addon's `contentFor()` hook for `type="tests-suffix"` is deprecated. Addons will no longer be allowed to inject arbitrary javascript here. Provide utilities that users can import into their own test setup code instead.

#### vendor-prefix

Returning content from an addon's `contentFor()` hook for `type="vendor-prefix"` is deprecated. Addons will no longer be allowed to inject arbitrary javascript here. If you really need to run script (non-module) code, provides your own script via your addon's `/public` directory and either document that app authors should createa a `<script>` element in their HTML that includes it, or use `contentFor()` with one of the `type`s that appears in `index.html` to emit the scrip tag automatically. (`contentFor` targeting HTML is not deprecated, this deprecation only covers targeting javascript bundles.)

#### vendor-suffix

Returning content from an addon's `contentFor()` hook for `type="vendor-suffix"` is deprecated. Addons will no longer be allowed to inject arbitrary javascript here. If you really need to run script (non-module) code, provides your own script via your addon's `/public` directory and either document that app authors should createa a `<script>` element in their HTML that includes it, or use `contentFor()` with one of the `type`s that appears in `index.html` to emit the scrip tag automatically. (`contentFor` targeting HTML is not deprecated, this deprecation only covers targeting javascript bundles.)

## Drawbacks

This is a change to the v1 addon API. V2 addons already cannot use the contentFor hooks this RFC aims to deprecate. One could argue against bothering to change the v1 addon API.

## Alternatives

We could leave this API alone, on the assumption that it will get included in a wider "deprecate v1 addons" RFC. I'm not advocating that because I don't think it's practical to deprecate v1 addons any time soon.

We could include `app-boot` in this RFC. It certainly deserves to be deprecated, and has only a single use in the ecosystem (ember-cli-fastboot). But I think it's proper replacement should get designed in a "v2 app format" RFC instead, so that the booting of an app (or a test suite) is codified clearly as code the user controls. 
