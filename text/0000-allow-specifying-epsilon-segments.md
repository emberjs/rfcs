- Start Date: 2017-01-26
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary
`One paragraph explanation of the feature.`

Allow for specifying of epsilon segments in the Ember Router DSL. This would
allow for the ability to have a nameless route.

Reference constraint 2.1 in http://www.nathanhammond.com/route-recognizer for
more information about epsilon segments.

# Motivation
`Why are we doing this? What use cases does it support? What is the expected
outcome?`

Having consulted Nathan Hammond, while he was working at LinkedIn, if a nameless
route would cause any issues, he suggested that the path forward to ensure the
engines are mounted to the correct route properly was for Ember to specify
epsilon segments.

As the release of isolated engines is coming up it becomes more apparent that
having a nameless route is a feature to consider. One of the main reasons is for
an engine to be mounted to a path other than its namespace. This will allow the
ability to use code from another engine and apply it to a route of another.
My use case is for supporting legacy URLs while moving and organizing code to a
different engine that makes more sense functionally. Especially to keep a
consistent UI experience across the application.

# Detailed design
`This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.`

Example:

Let's say that we have the following paths tied to the `foods` engine.

application/router.js
```js
this.mount('foods', { path: '/foods/view', resetNamespace: true });
this.mount('foods', { path: '/foods/preferences', resetNamespace: true });
this.mount('foods', { path: '/foods/saved', resetNamespace: true });
this.mount('foods', { path: '/foods/search', resetNamespace: true });
this.mount('foods', { path: '/foods/:query', resetNamespace: true });
```

Additionally, there is a `search` engine that supports other routes (e.g. `drinks`).
Currently, all the food search code lives in the `foods` engine which doesn't make
a holistic search experience across the application. Moving forward, I want to
move the food search code to the `search` engine while supporting the legacy
`foods/search` and `foods/:query` urls.

Clean urls (SEO)
- https://example.com/foods/pickles
- https://example.com/foods/foods-in-san-francisco
- https://example.com/foods/pizza-in-san-francisco
Dirty Urls
- https://example.com/foods/search?location=san%20francisco
- https://example.com/foods/search?keyword=pizza

In order to migrate the code and support legacy URLs there would need to be a
way to have a nameless route.

application/router.js
```js
this.route('foods', { path: '/foods', resetNamespace:true }, function setJobsRoutes() {
  this.mount('foods', { path: '/', resetNamespace: true });
  this.mount('search', { path: ':query', resetNamespace: true });
})
```

The above legacy URLs would mount the search engine when hitting the `foods/:query`
and `foods/search` paths. Within the search engine there would be a route called
`foods` which won't actually have a route, hence the nameless route.

# How We Teach This
`What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

How should this feature be introduced and taught to existing Ember
users?`

Once epsilon segments have been included, the Ember docs will need to be updated
to reflect this. Those changes should be relatively straightforward as shown in
the example above. The only thing that is uncertain is how the epsilon segments
would score to deserialize the route (mentioned in the unresolved questions).


# Drawbacks
`Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

There are tradeoffs to choosing any path, please attempt to identify them here.`

As referenced in Nathan Hammond's Blog above, segments are ranked in terms of their
specificity. Introducing epsilon segments would cause collisions for the final
score to deserialize the route.

# Alternatives
`What other designs have been considered? What is the impact of not doing this?`

Moving engine route logic into the application router logic.

For example:
application/router.js
```js
this.mount('foods', { path: '/foods/view', resetNamespace: true });
this.mount('foods', { path: '/foods/preferences', resetNamespace: true });
this.mount('foods', { path: '/foods/saved', resetNamespace: true });
this.mount('foods', { path: '/foods/applied', resetNamespace: true });
this.mount('search', { path: '/foods/:query', resetNamespace: true });
```

In the example above, the `foods` engine will mount if the route file matches
the first four paths and mount the `search` engine with dynamic segments. This
isn't ideal because its repetition of engine mounting to paths. Additionally,
this would require every engine route to be specified in the application router
file which is against how the two router files are intended to be used.

# Unresolved questions
`Optional, but suggested for first drafts. What parts of the design are still
TBD?`

Should epsilon segment scores count? E.G. should the score be '1' and not an
empty string how it is currently. This would move the segments scores up
('4' for static, '3' for dynamic, '2' for glob, '1' for epsilon).
