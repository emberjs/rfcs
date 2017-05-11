- Start Date: 2016-01-16
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

A Promise-aware rendering engine allows users to declaratively write synchronous looking code in their templates and have the template re-render when the promise is resolved.

# Motivation

A common point of frustration when using Ember Data relationships is the fact that they don't often re-render as expected.

A common example is the following. A `Book` belongs to an `Author`, but the `Author` may be null. This relationship is async:

```javascript
// app/models/book.js
import DS from 'ember-data';

export default DS.Model.extend({
  name: DS.attr(),
  author: DS.belongsTo('author'. {async: true})
});
```

Assuming that `book` is a `Book` model in the following code:

```handlebars
{{#if book.author}}
  <span>Yay, there is an author: {{book.author.name}}!</span>
{{else}}
  <strong>There's no author! Please input an author name and we'll figure it out.</strong>
  {{input value=form.author}}
{{/if}}
```

Even if the `book.author` promise resolves with `null`,  the rendered output will state that there is an author every time, which is innacurate. In addition, there's no way to know if the promise is resolving. This is also accurate for any promise-like object (jQuery, RSVP, native Promise, etc).

In the past, we have used `PromiseProxy`, which wraps a promise and proxies property look-ups to its `content`, which is a promise. This doesn't work because (fill this in when you can remember), and the implementation in Ember itself has known to be error-prone. This would also require the user to wrap all their promise objects in PromiseProxies, which feels unintuitive and repetitive.

# Detailed design

When Glimmer encounters a promise-like object, it should call `then`, and re-render the node when the promise is resolved or rejected. 


# Drawbacks

* Uses duck-typing to determine if something is a promise by calling `then`
* Could potentially be slow
* Potentially backwards incompatible.
* No way to inspect promise state unless helpers are provided (see below).

# Alternatives

An alternative without changing glimmer itself is to provide a set of helpers that allow the state of a promise to be handled by the user.

```handlebars
{{#await model.author as |author, error, isPending|
  {{#if isPending}}
    Loading...
  {{else if error}}
	Error! {{error}}
  {{else}}
	{{#if author}}
	  Yay! The author is: {{author.name}}
	{{else}}
	  Boo! There is no author!
	{{/if}}
  {{/if}}
{{/await}}
```

The syntax is a bit verbose, and doesn't feel as intuitive.

# Unresolved questions
