- Start Date: 2015-06-06
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Adds an easy way to add redirects to Ember's Router.

# Motivation

URLs should never break. When URLs change, they should be properly redirected
and be supported forever. However, redirects in Ember are clunky and don't
encourage developers to add and support redirects.

The best solution here would be something that is simple to use and avoids
URLs ever breaking.

# Example

```js
Router.map(function() {
  this.route('foo', { redirectFrom: '/bar' });
  this.route('baz', { redirectFrom: '/bat' }, function() {
    this.route('boo');
    this.route('bit', { redirectFrom: ['/bit', '/biz'] });
  });
});

// Generated Redirects:
// /bar     => /foo
// /bat     => /baz
// /bat/boo => /baz/boo
// /bat/bit => /baz/bit
// /bit     => /baz/bit
// /biz     => /baz/bit
```

# Detailed design

The design here come primarily from Jekyll's [redirect from](https://github.com/jekyll/jekyll-redirect-from)
plugin.

The basic idea is that routes can add a `redirectFrom` option which accepts
either a string or an array of strings that map **source** urls to **destination** urls.

In it's most basic from it looks like this:

```js
this.route('foo', { redirectFrom: '/bar' });
// /bar => /foo
```

But can also be specified as an array:

```js
this.route('foo', { redirectFrom: ['/bar', '/baz'] });
// /bar => /foo
// /baz => /foo
```

You'll notice that the **source urls are absolute**. This is to avoid any
potential issues as urls change over time. Meaning once a redirect is set, it
never has to be updated again.

### Nested Routes

Nested routes can often get redirected all at once. This can be represented as
an option on the parent route.

```js
this.route('foo', { redirectFrom: '/bar' }, function() {
  this.route('baz');
  this.route('bat');
});
// /bar/baz => /foo/baz
// /bar/bat => /foo/bat
```

Nested redirects work they same as if they were not nested.

```js
this.route('foo', { redirectFrom: '/bar' }, function() {
  this.route('baz');
  this.route('bat', { redirectFrom: '/bit' });
});
// /bar/baz => /foo/baz
// /bar/bat => /foo/bat
// /bit     => /foo/bat
```

Duplicates throw errors.

```js
this.route('foo', { redirectFrom: '/bar' }, function() {
  this.route('baz', { redirectFrom: '/bar/baz' });
  // Error
});
```

### Dynamic Segments

Dynamic segments for the destination url must be mapped from the source url.

If a segment of the source url is mapped to the destination url, then the url
should reflect it.

```js
this.route('foo', { path: '/foo/:foo_id', redirectFrom: '/bar/:foo_id' });
// /bar/1 => /foo/1
```

Same goes for wildcards.

```js
this.route('foo', { path: '/foo/*wildcardName', redirectFrom: '/bar/*wildcardName' });
// /bar/baz => /foo/baz
```

If a segment of the **source** url is not mapped, then that segment should be dropped.

```js
this.route('foo', { path: '/foo', redirectFrom: '/bar/:bar_id' });
// /bar/1 => /foo
```

If a segment of the **destination** url is not mapped, then it should throw an error.

```js
this.route('foo', { path: '/foo/:foo_id', redirectFrom: '/bar/:bar_id' });
// Error
```

### Redirecting Existing Routes

You may not redirect a url that still exists. This should throw an error.

```js
this.route('foo', { redirectFrom: '/bar' });
this.route('bar');
// Error

this.route('foo', { path: '/foo/:foo_id', redirectFrom: '/bar/:foo_id' });
this.route('bar', { path: '/bar/:bar_id' });
// Error
```

### Redirect Loops

Loops of redirects should be detected if possible and throw errors, most of
this should be covered by the "Redirecting Existing Routes" section above.

# Drawbacks

Unsure of any at this time.

# Alternatives

## this.redirect();

```js
Router.map(function() {
  this.redirect('/bar', '/baz');
});
```

The problem with this method is that it keeps redirects and routes separated
which would be easy to lose track of over time. Tying routes to their previous
revisions ensures that they get maintained.

# Unresolved questions

- How should dynamic segments be prioritized?
- How should we deal with mapping query parameters?

