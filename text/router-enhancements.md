- Start Date: 2016-07-18
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

The features of Ember's router have been unchanged since pretty much the initial
release.

This RFC focuses in twow features:

- Optional segments
- Dynamic segment constraints

These features can enable some usages that right now are cumbersome or require code duplication.

# Motivation


### Dynamic segments

Consider an e-commerce or marketing site nuilt in Ember that wants to leverage server-side rendering
with fastboot to be indexed in search engines.

Search engines are smart enough to index information based on the language, but it's common practice
to namespace content by language and/or country. Sometimes even the items displayed depeds on
the region, but the page itself is identical.

One real-world example of this is Apple's page.

The page version for the UK is `https://www.apple.com/uk/mac`, while the spanish version is in
`https://www.apple.com/es/mac`.

So far, this route can be expressed in the current's version of the router as `/:regionCode/mac`.

However, the US/International/Default version of the page is also available in `https://www.apple.com/mac`.
The content and layout of the page is the same in the three routes and yet as of today, implementing
this kind of pattern requires to duplicate the entire routes/controllers under two different
places. The template could be reused by overriding the `render` hook in the route, but still this
is awkward.

Ideally the user should be able to define the initial segment as optional so the router maps all those
urls to the same route giving.

Regional/Locale is just one usage of optional segments that is particularly common but not the only.

### Dynamic segments' constraints

Another feature that can ease the development is determine if a pattern matches based on some
condition that cannot be expressed solely with the microsyntax that the router provides.

Consider a weather app that displays the forecast for tomorrow with temperatures, and the user gets
to choose among a Celcius, Fahrenheit and Kelvin degrees, and that lives in the url.

This can be encoded on this string: `/forecast/:scale`. That entry would match `/forecast/celsius`,
`/forecast/fahrenheit` but also URLs like `/forecast/rankine` which is a unsupported scale.

Currently there is two solution for this, either codify these three routes as static routes
without dynamic segments, which would leave us with a similar duplication problem as the one
described for optional segments, or guard agains invalid scales in the route and perform a redirect
via `replaceWith` to the not-found page. However, taking lazy-loading engines into consideration,
handling constrains manually in the routes requires the app to load and boot the engine in order to
determine something that could be done much before and save all this expensive work.

Backend fragments like Ruby on Rails get a DRY solution to this allowing to receive an executable
piece of code that the dynamic segment has to validate in order of that route to match.

In the previous example the constraint would be that the scale is one of our whitelisted values, and
any other would cause the route to miss.


# Detailed design


### Optional segments.

Mimicking rails microsintax, I propose optional segments to be enclosed between parenthesis. The
example of the Apple store would be encoded

```
Router.map(function() {
  this.route('product', { path: '(/:region)/:productName' });
});


router.serialize('/mac'); // => routeName='products', params: { productName: 'mac' }
router.serialize('/uk/mac'); // => routeName='products', params: { productName: 'mac', region: 'uk' }
```


### Dynamic segment's constrains

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

  # Equivalent to:
  # this.route('forecast', {
  #   path: 'forecast/:scale',
  #   constraints: {
  #     scale: function(scale) {
  #       return /(celsius|fahrenheit|kelvin)/.test(scale);
  #     }
  #   }
  # });
});
```

# How We Teach This

Expand the guides on the routing topic to cover this use cases.

# Drawbacks

It adds some extra complexity to the routes for a feature that we've survived without for years.

Particularly the dynamic segment constraints, without considering the case of lazy-loaded engines,
can be emulated without too much effort by manually by checking the validity of the params in the
routes, and adds some vector of complexity since it's not clear which approach to use as they
admitely overlap.

# Alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions

- Do optional segments have a default value?
- What arguments should the constraint function receive apart from the dynamic segment itself?
- In routes with more than one dynamic segment, can constraints perform checkings on all segments
at once? Per example:

```js
const allowedScales = ['celsius', 'fahrenheit', 'kelvin'];

function isAllowedScale(scaleName) {
  return allowedScales.includes(scaleName);
}

Router.map(function() {
  this.route('forecast', {
    path: 'activities/from/:from/to/:to',
    constraints: function(from, to) {
      return moment(from).isBefore(moment(to));
    }
  });
});


router.serialize('/activitis/from/2016-04-11/to/2016-05-20'); // => matches
router.serialize('/activitis/from/2016-06-11/to/2016-05-20'); // => fails to match
```
