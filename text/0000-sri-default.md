- Start Date: 2015-07-10
- RFC PR: (leave this empty)
- ember-cli Issue: (leave this empty)

# Summary

Enable [Subresource Integrity [SRI]](http://www.w3.org/TR/SRI/) checks by default.

# Motivation

To promote the use of SRI in Ember apps as a safe default. Applications should be built with integrity attributes when it is safe to do so. (Unfortunately the main advantage won't be met by default, however confirming one attribute will)

This solves having poisoned CDN content: [An introduction to JavaScript-based DDoS](https://blog.cloudflare.com/an-introduction-to-javascript-based-ddos/)


# Detailed design

Install [ember-cli-sri](https://www.npmjs.com/package/ember-cli-sri) by default.

- Applications with relative paths will get SRI.
- Applications with `SRI.crossorigin` will get SRI on `fingerprint.prepend` assets
- Applications with `fingerprint.prepend` and `origin` specified and matching get a `SRI.crossorigin` of anonymous on `fingerprint.prepend` assets

By default development environments wont run SRI for performance reasons.

Further explanation available in: [ember-cli-sri](https://www.npmjs.com/package/ember-cli-sri)

# Drawbacks

- SRI won't always be on for sites with prepend due to SRI requiring CORS.
- CORS requirement adds a barrier to entry to some users.
- Broken SRI attrs would break the application.

# Alternatives

No other alternatives appear suitable.

# Unresolved questions

- Adding origin attribute to add a safe same-origin check that doesn't need CORS.
- Could users be warned until they explicitly set `SRI.enabled = false` or `SRI.crossorigin = `?
