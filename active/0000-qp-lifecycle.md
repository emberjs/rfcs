- Start Date: 2016-12-14
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Formalize the lifecycle of query-params.

# Motivation

The [guides for query-params](https://guides.emberjs.com/v2.10.0/routing/query-params/)
discuss at length how to set query-params, including customizations. They do
not, however, formally specify the _lifecycle_ of query-params. Specifically,
the following are left unspecified:

 * at what point in a transition are query-params synced from the Location
   to the Controller?
 * at what point in a transition are query-params synced from the Controller
   to the Location?

[This ember-twiddle](https://ember-twiddle.com/fee6f09acd62b0b567378b81dc76d4fc)
demonstrates some oddities:

 1. Load `/` directly (by typing it into the location bar and pressing Enter).
    The value from the Controller is `"defaultValue"`, which is the
    value the Controller specifies as the default. The value from
    `transition.queryParams.q` is `null`. In the Route's `model` hook, the
    Controller's value is the default value.
 1. Load `/?q=something` directly. The value from the Controller is
    `"something"`. The value of `transition.queryParams.q` is still `null`.
    The value that the Route's `model` hook sees from the controller is
    `"defaultValue"`, meaning the query-param hasn't been synced to the
    Controller.
 1. Load `/?q=forbidden` directly. This acts just like the `/?q=something`
    case. But if you click the `/?q=something` link in the footer and then
    the `/?q=forbidden` link, it changes. The `serializeQueryParam` kicks
    in, changing `"forbidden"` to `null`, which affects the location
    and the value from the Controller. This shows that `serializeQueryParam`
    is invoked on every transition _after_ the first.
 1. Load `/?v=123` directly. Because `v` isn't declared as a query-param,
    Ember strips it from the URL. This shows that _invalid_ query-params, but
    not _valid_ ones are re-serialized during the first transition.

# Detailed design

I do not yet have a clear, holistic implementation in mind. I think the
following requirements are a bare minimum:

 1. Define when `transition.queryParams` are available and what effects
    modifications to that object will have.
 1. Serialize all query-params back to the Location some time during the
    first transition just like all other transitions.
 1. Define where in the transition (e.g. "before `beforeModel`" or "between
    `afterModel` and `setupController`") query-params will be synced between
    the Location and the Controller.
 1. Allow routes to manipulate query-params at some point during the
    `beforeModel/model/afterModel/setupController` flow.

# Use-Cases

 * A query-param that lets admin users see other users' accounts. The
   application doesn't know whether the current user is an admin until after
   the session is established (say, `route:application#model`). If a non-admin
   goes to `/some-path?userId=3456`, the route should _clear_ that query-param
   after it is established the user lacks permissions.
 * A query-param that represents some split-testing state. The user navigates
   to `/some-path` and the application generates a split-testing ID on-the-fly,
   then sets it as a query-param. The Location should _immediately_ reflect
   this ID. (Having this ID in the URL could make it easier when handling
   support tickets since it would be clear which version of the interface
   they were using.)

# Drawbacks

Unknown

# Alternatives

[ember-query-params](https://github.com/knownasilya/ember-query-params/)
attempts to move some of the query-param logic into a service. This is good
for encapsulating logic around validating, filtering, and manipulating
query-params, but doesn't address the lifecycle issues.

# Unresolved questions

 * When are query-params synced from Location to Controller?
 * When are query-params synced from Controller to Location?
 * When should they be?
