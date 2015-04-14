- Start Date: 2015-04-13
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Currently, query parameters set to reloadModel will trigger a relevant `LoadingRoute` when changed.  Given the most common uses of query params, which are state-changing but without general context switching, like filters, it makes more sense to simply set the model to the resolving promise, allowing more localized loading displays, and a less jarring filtering experience.

# Motivation

This allows for a more graceful default behavior when query-params are used to reload a route's model.  Without this change, any application that has query params used to refresh a model have their filters hidden during the filtered request, or cannot use the LoadingRoute feature of Ember.

# Detailed design

This should be a fairly simple change.  Right now the `Route` `queryParams` hash allows query-param keys to provide options.  In those options, if `reloadModel` is set to true, when the query param is changed, the full route transition is triggered, which gives any `LoadingRoute` in the hierarchy above the current route the opportunity to be displayed until the model is fulfilled.

Instead of a full route refresh, I think fewer of the route's initialization steps should be triggered.  I'm not really familiar with what steps are in this process, but I think for someone familiar with it, it should be easy enough to find.

# Drawbacks

If there are people who have Ember apps that have query-params that `reloadModel` and rely on a `LoadingRoute` to display loading indication during that request, and had no other way of representing the model in a loading state, their template would have a period where it had no model.

# Alternatives

The query params hash on the route could provide an additional option flag to enable this feature, avoiding any breakage.

App creators can simply not use the LoadingRoute feature, but that seems like a shame, since it's a useful tool.

App creators can manage their model on the controller, to avoid the `reloadModel` default behavior.

# Unresolved questions

What part of the redirection would have to be changed?:

Would this be a change in the the route? Somewhere else? I really don't know many Ember internals are implemented.:x
