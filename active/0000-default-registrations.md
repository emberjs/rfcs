- Start Date: 2015-02-15
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Provide a mechanism by which a library can provide a *default* implementation
of an entry in the `container`, with easy overriding.

# Motivation

In Ember-I18n, I try to provide many hooks that the user can customize. For
example, if the `I18n` needs to find the plural form of a number, it calls
`I18n.pluralForm`. That function is part of the public API so people can
redefine it as necessary.

In a containerized world, I would prefer to have the `I18nService` find the
implementation via `this.container.lookup` or similar.

If the default implementation of that function is defined in
`addons/utils/plural-form.js`, then its natural container lookup call is
`this.container.lookupFactory('ember-i18n@util:plural-form'). The end-user
*could* unregister `'ember-i18n@util:plural-form'` and re-register
their own implementation, but that feels like digging too deep into the
internal details of Ember-I18n.

Instead, I would use a friendlier lookup path like `i18n:plural-form`. In order
to support that, I need to add an initializer that calls `register`:

```js
export default {
  name: 'ember-i18n',
  initialize: function initializeI18n(container, application) {
    var pluralForm = container.lookupFactory('ember-i18n@util:plural-form');
    container.register('i18n:plural-form', pluralForm, { instantiate: false });
  }
};
```

Then, to override that implementation, the user has to *unregister* the
default version and register their own:

```js
export default {
  name: 'my-i18n',
  initialize: function initializeI18n(container, application) {
    container.unregister('i18n:plural-form');
    container.register('i18n:plural-form', myPluralForm, { instantiate: false });
  }
};
```

I would like to be able to define a *default* implementation that users could
override without removing my default.

# Detailed design

I see three options:

### `container.forward`

```js
container.forward('i18n:plural-form', 'ember-i18n@util:plural-form');
```

Forwarding works like [`ln(1) -s`](http://linux.die.net/man/1/ln). In cases
where `singleton` is `true`, it returns the same object. If the upstream
lookup changes, so does the downstream. It handles `lookup` and `lookupFactory`.

### `{ default: true }`

```js
export default {
  name: 'ember-i18n',
  initialize: function initializeI18n(container, application) {
    var defaultPluralForm = container.lookupFactory('ember-i18n@util:plural-form');
    container.register('i18n:plural-form', defaultPluralForm, {
      instantiate: false,
      default: true
    });
  }
};
```

This would work like [Sass's `!default`](http://sass-lang.com/documentation/file.SASS_REFERENCE.html#variable_defaults_). Users can just define their own over the default
and Ember won't complain. But once a non-default entry was set (or once the entry
looked up via `lookup` or `lookupFactory`), it couldn't be "freely" overridden.

### Default Until Lookup

Treat *all* entries as if they had `{ default: true }` from the previous section.
Any entry can be freely overriden until `lookup` or `lookupFactory` is called on
it.

# Drawbacks

 * Bookkeeping cost in the container or registry.
 * Users might override a default inadvertently. For example, a copy-paste error
   could cause the user to override the *wrong* default, resulting in confusion.

# Alternatives

Three alternative implementations discussed above.

# Unresolved questions

 * Which version is least confusing to end-users?
