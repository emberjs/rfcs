- Start Date: 2016-07-26
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Add a base model at `app/models/-base.js` which is used to extend all the
applications' models from.

There is already the concept of an application wide adapter and serializer, so
this concept of a base class which all sub-classes extend from is made
available for models as well.

# Motivation

Currently if the models of your app have common functionality, a commong `attr`
or something similar, you need to create custom base model and make sure you
extend all existing models from it and update all models generated via the
`ember generate model` blueprint, so they extend from the same base model.

This is cumbersome, error prone and should be handled by Ember Data out of the
box.

Examples are `attr`s like `createdAt` and `updatedAt` available on all models;
or `revision: attr('string')` used in combination with a CouchDB backend.

# Detailed design

Create a base model at `app/models/-base.js` and update the model blueprint so
it doesn't extend from `DS.Model` but from the new base model instead.

# How We Teach This

Since this is a similar to the currently existing concept of an application
wide adapter and serializer, this doesn't introduce a new concept per se and
we can extend the documentation by the new - model - facet.

# Drawbacks

`¯\_(ツ)_/¯`

We can't be consistent with the adapter and serializer naming, as a model named
`application` might exist in the app and this would mean that all other models
would extend from this, which might lead to unexpected behavior.

# Alternatives

- create a mixin which is used instead of a base model and manually mix it into
  all your models
- create a base model yourself, and have the advantage of naming it as you like
  (`-base` might not be the best name)

# Unresolved questions

- what should the name of the base model be?
  - is `app/models/-model.js` better?
  - should the base adapter and serializer be renamed to
    `app/{adapter,serializer}/-base.js` too?
- should we make sure that the base model doesn't end up as specific model, so
  `store.findRecord('-base')` doesn't work?
