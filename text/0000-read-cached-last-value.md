- Start Date: 2020-08-18
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/656
- Tracking: 

# Read Previous Cached Value by Convention from within @cached Getters

## Summary

Allow a conventional way for RFC 566's @cached getters to retrieve
the last-returned value from within the next execution of the getter
to reduce computation load for complex results that are incremental
in nature, such as this example of an online auction where each
participant can see the bids sourced from the others:

```js
import { tracked } from '@glimmer/tracking';
import { cached, CACHED } from '...source module?';

class OnlineItemAuction {
  /**
   Bidders who have joined this particular item auction - this array
   is set from elsewhere and grows item by item over time.
   */
  @tracked bidders = [];
  
  /**
   Each bidder becomes a source of bids for this auction
   with history that is displayed locally for the duration
   of the auction and which initially comes from the server
   which is expensive, and so this array result should
   not fully regenerate, but incrementally regenerate instead
   as new bidders arrive.
   */
  @cached
  get bidSources() {
    // Read the previously returned set of bid sources
    // so that only new ones need be generated for newly
    // arrived bidders
    const _prevs = this[CACHED].bidSources || [];
    const bidders = this.bidders;

    // Generate a BidSource from each bidder, but avoid doing
    // it more than once per bidder.
    return bidders.map(bidder => {
      return _prevs.find(prev => prev.bidder === bidder) ||
        // expensive as it downloads initial state from server
        new BidSource(bidder);
    });
  }
}
```

## Motivation

This is not an uncommon use case: in particular, incremental arrays
of complex-to-create instances are very common, especially where
the feeder array (bidders) increments over time like above.

Despite this appearing at first glance to be a more advanced
use case, I have found empirically over the past 7 years
that this is one of a (shrinking) list of cases that arises quite
often and creates frustration among solid-JS but new-to-Ember
developers.
In this case, their experience causes recognition of the need to
memoize, but there is no solution by convention.  This seems
counterintuitive (to me as well) as frameworks like Ember are all
about optimizing computation and rendering of UI-bound data.

Perhaps more problematically, junior developers come up with some
pretty varied and hard-to-debug versions of this pattern and
that can really slow adoption and confidence (not to mention
code reviews and QA).

All levels would be helped tremendously by an out-of-the-box
conventional solution, and furthermore, this sits very well with
the @cached decorator concept (in fact, the implementation if
@cached is in place is 2 extra lines of code if done as above).

## Detailed design

A getter that is marked for memoization using `@cached` can access
its last-returned value by reading a Symbol-driven (or otherwise
conventional) property on its own instance using the same key
as the property's name.

At each read of the getter, which is intercepted already by
@cached, the value returned by the @cached descriptor's handing
is assigned to the object instance, more or less as follows:

### Updates to @cached getter wrapper

- Existing step: Read value to return from cache or getter as
  case may be
- __New step:__ Store value in a simple hash stored at
  `this[CACHED]` in a key matching the name of the property
  getter:  
  `this[CACHED][key] = value`
- Existing step: Return the value

### Resource Leaks

There is very little risk, but if necessary, `this[CACHED]` could
be wrapped in a WeakMap with one key (the instance) to break a
potential cycle if a cached value references its instance, but
that's generally a problem already - this doesn't make it worse.

Beyond that, the only overhead is the hash since the data itself is
presumably referenced elsewhere as well (i.e. garbage collection is
unlikely to be delayed on the data since it was gotten to be used).

## How we teach this

An extension of the `@cached` documentation is the natural place to
put this, using a strong and broadly-relatable anecdote
(like bidding in the example).
Building up an example of such a use-case from scratch would
certainly make the need for this feature very evident.

Assuming a knowledge of `@cached` (or any experience with it), it
doesn't take long to end up needing this additional feature as
`@cached` itself is designed to limit re-computation. Combine that
with an array-type property and you have a very likely demand
for this pattern.

Existing users are apt to pursue memoization topics if they have had
anything more than a trivial exposure to Ember (though perhaps
not by this unfortunately-less-than-colloqial name for a concept
that is itself quite practically simple).
Across the 20+ production-grade apps we've built on Ember, I can't
think of a single one where this wasn't important for performance
in at least some circumstance of the application.

No additional reorganization is needed for the guides.

## Drawbacks

Direct prior-cached-value-reading may come across as an exposed
internal working of the platform required to offset a design
flaw.
However this is a specialized case and therefore merits
a more specific solution.
Review of this RFC may lead to better ergonomics to address this.

Use of this as a crutch for an absence of better coding practices
may also arise, but this approach is not so ergonimic as to be
easier than "the right way" to handle a different use case.
In fact, it only applies if `@cached` is in use, limiting the
domain of potential misuse.

## Alternatives

- Let the developer continue to manage this on a case-by-case
  basis
- Provide a base class that implements this 'automagically',
  however that's exactly the opposite direction Ember and those
  like it are moving in.

## Unresolved questions

- Is this the best ergonomic implementation?
- Is there an issue with the possibility that one getter can access
  another getter's prior value (since `this[CACHED]` can
  have any key read - is this actually a beneficial
  side-effect?)
