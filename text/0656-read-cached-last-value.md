- Start Date: 2020-08-18
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/656
- Tracking: 

# Read Previous @cached value with new @cacheFor Decorator

## Summary

Introduce the `@cacheFor(propName)` decorator to provide
a conventional way for RFC 566's `@cached` getters to retrieve
the last-returned value from within the next execution of the getter,
enabling computation load reduction for complex results that are
incremental nature. Consider this example of an online auction where
participants arrive slowly and each can see the bids sourced from the
others (showing the proposed `@cacheFor` feature):

```js
import { tracked } from '@glimmer/tracking';
import { cached, cacheFor } from '...';

class OnlineItemAuction {
  /**
   Bidders who have joined this particular item auction - this
   array is set from elsewhere and grows item by item over time.
   */
  @tracked bidders = [];
  
  /**
   This (read only) property returns the most recently-returned
   result of the bidSources getter below, or an initial value
   if the getter has not been called yet. This property is not
   useful to be bound elsewhere - it is 'volatile' in that it is
   calculated on demand when called by simply reading the last
   value in the cache for the desired property (or the initial
   value provided).
   */
  @cacheFor('bidSources') _bidSources = [];

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
    const _prevs = this._bidSources;
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
conventional solution, and furthermore, this sits very well on top
of the @cached decorator concept (in fact, the implementation if
@cached is in place is a few extra lines of code if done as above).

## Detailed design

A getter that is marked for memoization using `@cached` can access
its last-returned value by reading a property on its own instance
identified with the proposed `@cacheFor(propName)` decorator.

The `@cacheFor(propName)` decorator is itself a volatile getter
that simply returns the last value be reading the cache. This is
done on demand and no state is stored in the getter itself. If the
cache is not found to exist yet for the related cached property,
then the initial value (if provided) is returned instead.

> To Do: Put in sample implementation code later today

### Resource Leaks

This is a very clean and lightweight solution - the cache exists
already (or not, if never created), but in either case, this
decorator does not create additional resource load or references.

## How we teach this

An extension of the `@cached` documentation is the natural place to
put this, using a strong and broadly-relatable anecdote
(like bidding in the example).
Building up an example of such a use-case from scratch would
certainly make the need for this feature very evident.

Assuming a knowledge of `@cached` (or any experience with it), it
doesn't take long to end up looking for the proposed functionality as
`@cached` itself is designed to limit re-computation. Combine that
with an array-type property and you have a very likely demand
for the pattern addressed by `@cacheFor(propName)`.

Existing users are apt to pursue memoization topics if they have had
anything more than a trivial exposure to Ember.
Across the 20+ production-grade apps we've built on Ember, I can't
think of a single one where this wasn't important for performance
in at least some circumstance of the application.
Unfortunately, implementation of solutions for this issue can be
highly varied as Ember (currently) leaves incremental state as
a matter for the end-developer to address.

No additional reorganization is needed for the guides.

## Drawbacks

Use of this as a crutch for an absence of better coding practices
may arise, but this approach is not really beneficial to other cases.
As proposed, it only applies if `@cached` is in use, limiting the
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
  another getter's prior value (since the prior value can
  be read anywhere - is this actually a beneficial
  side-effect?)
