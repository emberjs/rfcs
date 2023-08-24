---
stage: discontinued
start-date: 2022-01-29T00:00:00.000Z
release-date: Unreleased
release_versions:
teams:
  - data
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/790'
  discontinued: 'https://github.com/emberjs/rfcs/pull/887'
project-link:
suite:
---

# EmberData | Deprecate ajax requests

## Summary

Deprecates use of `ajax` methods in favor of `fetch` to request API data.

## Motivation

Currently, `ember-data` maintains two methods for users to fetch data from their backend - fetching with `jQuery.ajax` and `fetch`.  Confusion further exists in the adapters if users implement functionality to change out of the box behaviour. They can override [`ajax`](https://github.com/emberjs/data/blob/e076e0ae71ae6426ca53ad3c5501a0af7ceca883/packages/adapter/addon/rest.ts#L1091-L1126) methods but without a clean interface to operate with, what they really may be doing is overriding how fetch works.

With Ember 4.0 dropping jQuery, we should take a hard stance on dropping the use of `ajax` in the next major release.

## Detailed design

The first step is putting in place a deprecation if an adapter uses `ajax` to make a request. Since `ember-data` switched the default behaviour to use fetch in 4.0, this deprecation will only apply to those who have overrided the default with `useFetch = true` in their adapters.

Second, we will expose fetch related methods on the adapters that are equivalent in use and flexibility as the current `ajax` options.  Note the [`minimum-adapter-interface`](https://github.com/emberjs/data/blob/master/packages/store/addon/-private/ts-interfaces/minimum-adapter-interface.ts) does not assert the method by which a request is made. As a result, how we design and implement these methods should not have an effect on those users who have implemented their own adapter.

In 5.0, all `ajax` methods and supporting infrastructure will be removed. The list of affected methods includes:

- ajax
- _ajaxRequest
- _ajax
- ajaxOptions
- _ajaxUrl

Moreover, we may rename methods not part of the minimum-adapter-interface. Moreover, we expect to implement the following methods with [fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch), although could be changed in the actual implementation.

- fetch
- fetchOptions
- fetchSuccess
- fetchError
- fetchUrl

## How we teach this

This is best taught through a deprecation guide. Users using `ajax` will need to install `ember-fetch` or rely on the default global `fetch` API provided by the browser.  This has some implications, for example, with how query params are serialized.  Users will have to be careful to ensure the behaviour of their API changes with any changes that result from switching to fetch.

## Drawbacks

Some user's technology stacks may be entirely dependent on `ajax` for requesting API data.  This isn't necessarily an easy switch and may require multiple improvements to various layers to use `fetch`. For those users, we can document how they can still implement their own adapter to use `ajax`.  This will involve overriding the existing `fetch` methods.  For example, if your API still needs to be serviced with `ajax` to perhaps take advantage of [ajax options](https://api.jquery.com/jquery.ajax/#jQuery-ajax-url-settings) or [ajaxPrefilter](https://api.jquery.com/jquery.ajaxprefilter/), simply override the existing `fetch` methods.  The goal of this refactor would be to ensure users who still need to use `ajax` has a happy path to doing so.

## Alternatives

- Continue providing and exposing public-ish adapters methods as `ajax`.
- Provide both `fetch` and `ajax` methods together in the adapters.
