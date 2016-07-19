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


### Optional segments

Consider an e-commerce or marketing site built in Ember that wants to leverage server-side rendering
with Fastboot to be indexed in search engines.

Search engines are smart enough to index information based on the language, but it's common practice
to namespace content by language and/or country. Sometimes even the items displayed depends on
the region, but the page itself is identical.

One real-world example of this is Apple's page.

The page version for the UK is [https://www.apple.com/uk/mac](https://www.apple.com/uk/mac), while the
Spanish version is in [https://www.apple.com/es/mac](https://www.apple.com/es/mac).

So far, this route can be expressed in the current version of the router as `/:regionCode/mac`.

However, the US/International/Default version of the page is also available in [https://www.apple.com/mac](https://www.apple.com/mac).
The content and layout of the page is the identical in the three URLS and yet as of today, implementing
this kind of pattern requires to duplicate the entire routes/controllers under two different
places. The template could be reused by overriding the `render` hook in the route, but still this
is awkward.

Ideally the user should be able to define the initial segment as optional so the router maps all those
URLs to the same route.

Region/Locale is just one usage of optional segments that is particularly common but not the only.

All segments can be optional, no matter globs, static or dynamic.

### Dynamic segments' constraints

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


# Detailed design


### Optional segments.

Mimicking rails micro syntax, I propose optional segments to be enclosed between parenthesis. The
example of the Apple store would be encoded

```
Router.map(function() {
  this.route('product', { path: '(/:region)/:productName' });
});


router.serialize('/mac'); // => routeName='products', params: { productName: 'mac' }
router.serialize('/uk/mac'); // => routeName='products', params: { productName: 'mac', region: 'uk' }
```

### Dynamic segment's constraints

Also stealing Ruby on Rails' syntax, the user can provide a constraint using the
config object passed as second argument.


```js
const allowedScales = ['celsius', 'fahrenheit', 'kelvin'];

function isAllowedScale(scaleName) {
  return allowedScales.includes(scaleName);
}

Router.map(function() {
  this.route('forecast', { path: 'forecast/:scale', constraints: { scale: isAllowedScale } });
});


router.serialize('/forecast/kelvin'); // => routeName='forecast', params: { scale: 'kelvin' }
router.serialize('/forecast/newton'); // => not recognized route.
```

Although the generic case of receiving a function that returns a boolean is the most generic one,
it might worth to add a RegExp support as a convenience.

```js
Router.map(function() {
  this.route('forecast', { path: 'forecast/:scale', constraints: { scale: /(celsius|fahrenheit|kelvin)/ } });

  // Equivalent to:
  // this.route('forecast', {
  //   path: 'forecast/:scale',
  //   constraints: {
  //     scale: function(scale) {
  //       return /(celsius|fahrenheit|kelvin)/.test(scale);
  //     }
  //   }
  // });
});
```

### Combination of optional segments with constraints

Both approaches can be combined. Imagine that in the weather forecast example, if the there is
no temperature scale specified, the app will detect the best one based on the location of the
device.

On that situation, `/forecast` is a valid URL and has to point to the same route as the others with
specific scale.

```js
const allowedScales = ['celsius', 'fahrenheit', 'kelvin'];

function isAllowedScale(scaleName) {
  return allowedScales.includes(scaleName);
}

Router.map(function() {
  this.route('forecast', { path: 'forecast(/:scale)', constraints: { scale: isAllowedScale } });
});


router.serialize('/forecast/celsius'); // => routeName='forecast', params: { scale: 'celsius' }
router.serialize('/forecast/newton'); // => not recognized route.
router.serialize('/forecast'); // => routeName='forecast', params: { }
```

In all cases the same `forecast` route is invoked, but the scale is not optional but the constraint
takes care that, if provided, the scale is a supported one.

# How We Teach This

Expand the guides on the routing topic to cover this use cases. For devs with Rails, Django or ASP.net
background this feature should look very familiar already.

# Drawbacks

It adds some extra complexity to the routes for a feature that we've survived without for years.

Particularly the dynamic segment constraints, without considering the case of lazy-loaded engines,
can be emulated without too much effort today by manually checking the value of the dynamic segments
in the routes, so this proposal adds some vector of complexity since it's not clear which approach
to use as they admittedly overlap.

# Alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions

- What arguments should the constraint function receive apart from the dynamic segment itself? The
route seems a reasonable request.
- In routes with more than one dynamic segment, can constraints perform checks on all segments
at once? Per example:

```js
Router.map(function() {
  this.route('forecast', {
    path: 'activities/from/:from/to/:to',
    constraints: function(from, to) {
      // This constraint takes both dynamic segments and performs an assertion
      // that takes both segments into consideration.
      return moment(from).isBefore(moment(to));
    }
  });
});


router.serialize('/activitis/from/2016-04-11/to/2016-05-20'); // => matches
router.serialize('/activitis/from/2016-06-11/to/2016-05-20'); // => fails to match
```

- How complex can constraints be? Just pure functions or can they have access to services? I think
that for simplicity they should start as simple as possible, and they could grow to support more
complex usages.
- When an optional dynamic segments are combined with constraints on that segment, and the segment
is not present, does the constraint function run with `undefined` as value or it doesn't run at all?
I lean towards the second.
