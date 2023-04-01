---
stage: # FIXME
start-date: 2023-03-31T00:00:00.000Z
release-date: # FIXME
release-versions: # FIXME
teams:
  - framework
prs: # FIXME
project-link: # FIXME
meta: # FIXME
---

# Setting dynamic query params in Controllers

## Summary

The current way developers bind query params to the Ember app is by defining the `queryParams` property on the Controller with hard coded values. It will be useful for developers who leverage a server-driven UI to dynamically bind query params to the Ember app.

## Motivation

With the current implementation of how query params work inside of an Ember app, developers are forced to have pre-knowledge of what query params are able to be binded to the app and hard code these values in the Controller. This may work for an app that is rigid in its interface, however there are use-cases where a server-driven UI can define the query params dynamically based on the user profile.

Take for instance a simple filter bar where the filter options are known in advance. Hard coding values can work for this use-case because there is a limited amount of options and these options are generic for all users.

```js
// controller/simple/filter-bar.js
queryParams = ['keywords', 'location', 'company']
```

Now let's expand this use-case and dynamically generate the simple filter bar into a complex filter, where the filter options are server-driven and tailored specifically to each user. The filter options are not known in advance and it will currently break how query params work in an Ember app.

```js
// controller/complex/filter-bar.js

// this will not work in Ember today
queryParams = this.model.filterOptions
```

## Detailed design

I propose that we enable Ember the option to dynamically bind query params based on server-driven requirements.

* Allow the Route's `setupController` to dynamically bind query params to the Controller's `queryParams` field
* Keep the current option to statically bind query params to the Controller's `queryParams` field

Example:

```js
// route/complex/filter-bar.js
setupController(controller, model) {
  controller.queryParams = model.filterOptions
}
```

In addition to enabling server-driven query params, this will unlock the ability for servers to generate query params using AI. For example the system has detected that the user profile is a Nurse and generates the list of filters tailored to the Nursing profession.

## How we teach this

Update the [Ember Docs](https://guides.emberjs.com) that contain the following passage:

_Note that you can't make queryParams be a dynamically generated property (neither computed property, nor property getter); they have to be values._ - [Ember query-params](https://guides.emberjs.com/release/routing/query-params/)

This will need to be changed to:

_You can bind query params dynamically using the Route's `setupController` to set the `queryParams` property to the controller_

```js
// route/index.js
setupController(controller, model) {
  controller.queryParams = model.queryParams
}
```

## Drawbacks

Dynamically setting query params could make the code less clear which query params are binded to the controller.

## Alternatives

We can decide not to do anything, but this leaves the problem where server-driven UI teams cannot scale at the same pace as Ember.

## Unresolved questions

---
