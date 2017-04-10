- Start Date: 2017-04-10
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Introduce a way to redirect browser to an external URL which isn't handled 
by current Ember application.

# Motivation

It's pretty common that Ember application is connected with another service 
or module that can't be easily integrated into Ember application. For a basic 
use-case we can bring an Ember application with an external login page. User 
should be redirected to that page when he enters a page with restricted access.
There can be other applications using the same login page so integrating it into 
Ember app can be complex. We cannot use `this.trasitionTo()` as it 
asserts if `routeName` is handled by Ember application: `The route ${targetRouteName} 
was not found`. 

In the past we're using `window.location.replaceWith()` to accomplish 
the goal. When Ember Fastboot was introduced this problem got more complicated as 
we need to write separate handling for server side. Introducing public API in Ember 
that allows to redirect to external URL will allow Ember FastBoot to add it's own 
handling for server side.

Old way of external redirecting:
```javascript
if (this.get('fastboot.isFastBoot')) {
  this.get('fastboot.response.headers').set('location', 'http://google.com');
  this.set('fastboot.response.statusCode', 302);
} else {
  window.location.replaceWith('http://google.com');
}
```

# Detailed design

New methods will be introduced on `Route` class for external redirects: 
`externalTransitionTo($url)` and `externalReplaceWith($url)`. Both methods implements
similar functionality to existing `transitionTo()` and `replaceWith()` - the main
difference is that new methods accepts URL that isn't defined in router.

# How We Teach This

This RFC suggest introducing new methods in Ember API. I think it'll be best to cover
new methods by adding new section in Guides article about [Redirecting](https://guides.emberjs.com/v2.12.0/routing/redirection/).

Short info in release notes with a link to updated Redirecting guide should be
sufficient for existing Ember users.

# Drawbacks

The only drawback I see is that it expands the surface of the API.

# Alternatives

Extending functionality of existing `transitionTo()` and `replaceWith()` methods:
1. Remove assert that check if url is defined in router - Previously we prevented
developers from making a bug using an url which isn't handled by th application.
Although it still can be a valid approach as Guides suggest using `routeName` 
instead of URL.
1. Add `allowExternal` flag in `options` parameter for existing `transitionTo()` 
and `replaceWith()` methods. 

# Unresolved questions

Should we additionally introduce `externalIntermediateTransitionTo()`? What is 
the use-case for this method?
