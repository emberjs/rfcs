- Start Date: 2016-07-18
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

The features of Ember's router have been unchanged since pretty much the initial
release.

This RFC focuses in two features:

- Optional segments
- Dynamic segment constraints

These features can enable some usages that right now are cumbersome or require code duplication.

# Motivation


### Optional Segments

Consider an e-commerce or marketing site built in Ember that wants to leverage server-side rendering
with Fastboot to be indexed by search engines.

Search engines are smart enough to index information based on the language, but it's common practice
to namespace content by language and/or country. Sometimes even the items displayed depends on
the region though the page itself is identical.

One real-world example of this is Apple's page.

The page version for the UK is [https://www.apple.com/uk/mac](https://www.apple.com/uk/mac) while the
Spanish version is in [https://www.apple.com/es/mac](https://www.apple.com/es/mac).

So far, this route can be expressed in the current version of the router as `/:regionCode/mac`.

However, the US/International/Default version of the page is also available in [https://www.apple.com/mac](https://www.apple.com/mac).
The content and layout of the page is the identical in the three URLS yet, as of today, implementing
this kind of pattern in Ember requires us to duplicate entire routes/controllers in two different
places. The template can be reused by overriding the `render` hook in the route, but still this
is awkward.

Ideally the user should be able to define the initial segment as optional so the router can map both URL patterns to the same route.

Region/Locale is just one usage of optional segments that is particularly common but not the only.

### Dynamic Segment Constraints

Another feature that can make some usages easier is adding runtime constraints that dynamic segments
must satisfy for the route to match and that cannot be expressed solely with the micro syntax that
the router provides.

Consider a weather app that displays the forecast for tomorrow with temperatures, and the user gets
to choose among a Celsius, Fahrenheit and Kelvin degrees, and preference is stored in URL.

This can be encoded on this string: `/forecast/:scale`. That entry would match `/forecast/celsius`,
`/forecast/fahrenheit` but also URLs like `/forecast/rankine` which is an unsupported scale.

Currently there is two solutions for this. Either to codify these three routes as static routes
without dynamic segments, which would leave us with a similar duplication problem as the one
described for optional segments, or guard against invalid scales in the route and perform a redirect
via `replaceWith` to the not-found page. However, taking lazy-loading engines into consideration,
handling constraints manually in the routes requires the app to load and boot the engine in order to
determine if this URL can be handled, and with route constraints that could be done eagerly by the routes
and avoid booting the engine at all.

Backend frameworks like Ruby on Rails get a DRY solution to this allowing to receive an executable
piece of code that the dynamic segment has to validate in order of that route to match.

In the previous example the constraint would be that the scale is one of our white-listed values, and
any other would cause the route to miss.


# Detailed Design


### Optional Segments

Mimicking the Rails microsyntax, I propose optional segments to be enclosed between parentheses. The
example of the Apple site would be encoded:

```js
Router.map(function() {
  this.route('product', { path: '/(:region)/:productName' });
});


router.serialize('/mac'); // => routeName='products', params: { productName: 'mac' }
router.serialize('/uk/mac'); // => routeName='products', params: { productName: 'mac', region: 'uk' }
```

All segments can be optional, no matter globs, static or dynamic.


### Dynamic Segment Constraints

Also borrowing from Rails' microsyntax, the user can provide a constraint using the
config object passed as second argument. For this proposal constraints may only be
regular expressions.

```js
Router.map(function() {
  this.route('forecast', {
    path: 'forecast/:scale',
    constraints: { scale: /(celsius|fahrenheit|kelvin)/ }
  });
});
```

### Combining optional segments with constraints

These new features can be combined. Imagine that in the weather forecast example, if there is
no temperature scale specified, the app will detect the best one based on the location of the
device.

In that situation, `/forecast` is a valid URL and would point to the same route as the others with
a defined scale.

```js
Router.map(function() {
  this.route('forecast', {
    path: 'forecast/(:scale)',
    constraints: { scale: /(celsius|fahrenheit|kelvin)/ }
  });
});


router.serialize('/forecast/celsius'); // => routeName='forecast', params: { scale: 'celsius' }
router.serialize('/forecast/newton'); // => not recognized route.
router.serialize('/forecast'); // => routeName='forecast', params: { }
```

In this example the `/forecast` route is matched even if the scale is ommited. Further, the scale constraint
is applied if there is any segment following `/forecast` to identify if it could possibly match
the optional dynamic path segment.

# How We Teach This

Expand the guides on the routing topic to cover these two new features. For devs with Rails, Django or ASP.net
background this feature should look very familiar already.

# Drawbacks

This adds additional complexity to the route microsyntax for a feature that we've survived without for years.

Particularly the dynamic segment constraints, without considering the case of lazy-loaded engines,
can be emulated without too much effort today by manually checking the value of the dynamic segments
in the routes. This proposal makes it less clear which approach
to use as they admittedly overlap.

# Alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions

None.