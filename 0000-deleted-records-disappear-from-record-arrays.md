- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

When a record is deleted, but not yet commited, on Ember Data, it's removed from
record arrays. I would like to change that: introduce a flag `isDeleting` and
remove a record from record arrays only when it's in the `deleted.saved` state.

# Motivation

The current behaviour makes it difficult to implement some of the common scenarios
and may introduce unintentional issues, which may be perceived as bugs.

One of such issues may occur when we have a list of items and a delete button.
When a delete button is clicked on a record, the record will disappear instantly.
This is a UX issue for me, because it looks like the record is already removed and
showing an indicator needs to happen in some other place (usually I would do it
somewhere in the record's element, most likely on a button itself).

If the request fails, a developer needs to decide what to do in such case, usually
either try again or rollback a record to show it on the list again (since it was
not deleted). This may look weird especially when the server responds quickly.
Showing the error message alone and not reverting record also feels like a bad
UX - the record is not actually deleted after all.

Each of this issues can be fixed, but it requires a non trivial solutions. In essence
it's much easier to filter out records with a certain state (for example
`isDeleting` if someone wants an old behaviour) than to show a list with
records that are removed from an underlying array.

# Detailed design

`deleted.uncommited` and `deleted.inFlight` should mark an `isDeleting` flag
instead of `isDeleted`. This alone will result in record not being deleted from
record arrays right away.

# Drawbacks

It's not backwards compatible.

# Alternatives

It's possible to implement an ArrayProxy wrapper over a record array and keep
not saved deleted records until they transition to `deleted.saved`.
