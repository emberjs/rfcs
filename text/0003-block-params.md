---
Start Date: 2014-08-18
RFC PR: https://github.com/emberjs/rfcs/pull/3
Issues:
  Ember Stream support: emberjs/ember.js#5522
  Handlebars parser support: wycats/handlebars.js#906
  HTMLBars compiler support: tildeio/htmlbars#147

---

# Summary

Introduce block parameters to the Handlebars language to standardize context-preserving helpers, for example:

```handlebars
{{#each people as |person|}}
  {{person.name}}
{{/each}}
```

# Motivation

### The Problem

There is no idiomatic way to write a helper that preserves context and yields values to its template. This is particularly painful for components which have strict context-preserving semantics.

### Current workarounds

- Don't write components that need to yield a value.
  - *Problem:* This may not be an option.
- Invent a non-standard per-helper syntax (like `{{#with foo as bar}}` or `{{#each item in items}}`) that hook into the undocumented `keywords` to inject variables.
  - *Problems:* Custom syntaxes are not in the spirit of the Handlebars language and require the consumer to know the special incantation. Component authors must an non-trivial understanding of how `keywords` work.

### New possibilities

```handlebars
{{#for-each obj as |key val|}}
  {{key}}: {{val}}
{{/for-each}}
```

```handlebars
{{#form-for post as |f|}}
  {{f.input "title"}}
  {{f.textarea "body"}}
  {{f.submit}}
{{/form-for}}
```


# Detailed design

- Phase 1: Add block params to the Handlebars language
- Phase 2: Rewrite Ember's helpers to accept streams
- Phase 3: Add block param support to `{{each}}` and `{{with}}`

### Phase 1: Add block params to the Handlebars language

The proposed syntax is `{{#x-foo a b w=x y=z as |param1 param2 ... paramN|}}` and is only available for block helpers.

The names of the block parameters are compiled into the inner template, but are not known to the helper (`x-foo` in the example above). To call a template and populate its block params we use the arguments option:


```javascript
var template = compile('{{person.name}}', {
  blockParams: [ 'person' ]
});

template({}, ..., [ personModel ]);
```

More commonly, block params will be defined inside of the template.

```
{{#with currentPost.author as |a|}}
  {{a.name}} <em>{{a.email}}</em>
{{/with}}
```

```javascript
registerHelper('with', function(object, options) {
  return options.fn(this, ..., [ object ]);
});
```

For compatibility reasons, the *number of block params* are passed to the helper so that the pre-block-params behaviour of the helper can be preserved. Example:

```javascript
function eachHelper(..., options) {
  if (options.blockParamsLength > 0) { /* do new behaviour */ }
  else { /* do old behaviour */ }
}
```

### Phase 2: Rewrite Ember's helpers to accept streams

In the `with` example above, if the `currentPost` changes the `a` block param should update. This means it's not sufficient to pass only the initial value of the author in the arguments. Instead, we pass a stream which emits values whenever the observed property changes.

In Handlebars, a block param can appear anywhere that an identifier can, for example `{{log a.name}}`. This means that all helpers would need to be modified to understand streams.

### Phase 3: Add block param support to `{{each}}` and `{{with}}`

Deprecate context-changing and ad-hoc keyword flavors of `{{each}}` and `{{with}}` in favor of block params.

# Drawbacks

- Handlebars already has a similar notion of with `data` which can lead to confusion.

# Alternatives

To my knowledge, no other designs have been considered. Not implementing this feature would mean that components would continue to be difficult to compose.

# Unresolved questions

The associated HTML syntax for HTMLBars needs to be finalized.
