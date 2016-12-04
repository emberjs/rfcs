- Start Date: 2016-12-04
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Fastboot app server should accept POST requests

# Motivation

When deployed within a larger system it is faster to have a server that
sits between the original request source (browser) and the fastboot
server embed the data payload into a `[POST] request.body` than have the
Fastboot server make an additional request to an external API server.

# Detailed design

Currently the Fastboot app server will make an external request for
required data. However, we can rely on Ember Data's store to server
the local record if it exists rather than make that external request.
When deployed to a larger system a server sitting in front of the
Fastboot server may have access to the data directly or a data cache.
The inbound `[GET]` request can be delegated to the Fastboot server as a
`[POST]` but with this payload as the `request.body`. This saves the app
server the additional external request.

Extremely high-yield systems will notice significant savings as the cost
of HTTP requests can add 10s of, if not 100s of, ms to the entire
request/response cycle.

Here is a visualization of what is currently possible:

![](http://i.imgur.com/JS2jYz2.png)

And here is with the ability to `[POST]` to the fastboot app server

![](http://imgur.com/lRZedIH.png)

This method is further helped with the Fastboot Shoebox and Ed
Faulkner's
[ember-data-fastboot](https://github.com/cardstack/ember-data-fastboot)

In our client application we have an `instance-initializer` that is
forced to run *before* `ember-data-fastboot`. This initializer makes use
of
[fastboot-filter-initializers](https://github.com/ronco/fastboot-filter-initializers)
so it is only included in the Fastboot build. We grab the request from
the Fastboot service, serialize the `body` and `store.push`. From here
`ember-data-fastboot` handles the rest by dumping the store into the
shoebox.

# How We Teach This

I suspect only those working in large systems would need this or even be
in a position to need this. The diagrams I've included above could be
useful in teaching the benefits. The initializer I described above is
about 10 LOCs. An example initlizer could be written, or maybe this is
something that is included with the Fastboot addon and you can simply
opt-into it.

# Drawbacks

This does add a new code path within the Fastboot app server, but beyond
that I'm open to ideas on what other potential drawbacks there could be.

# Alternatives

The only alternative I know about is the current method of making the
additional external request within the Fastboot render cycle.

# Unresolved questions

I'm interested in having the merit of this feature vetted and maybe
there are more optimizations for this scenario that we overlooked.
