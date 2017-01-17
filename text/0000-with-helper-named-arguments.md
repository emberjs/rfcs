- Start Date: 2017-01-17
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Allow the `with` helper accept named arguments (key/value pairs similar to the hash syntax):

```handlebars
{{#with
  topLeft     = (some-math a b c)
  topRight    = (some-math b a c)
  bottomLeft  = (some-other-math b d)
  bottomRight = (some-math c a b)
  center      = (some-other-math d d)
  focus       = (some-other-math d a b)
  foo         = "Bar"
}}
  <p>Top left corner is: {{topLeft}}</p>
{{/with}}
```


# Motivation

When there's a need to define multiple local variables in a template, multiple positional arguments are used:

```handlebars
{{#with
  (some-math a b c)
  (some-math b a c)
  (some-other-math b d)
  (some-math c a b)
  (some-other-math d d)
  (some-other-math d a b)
  as |topLeft bottomLeft bottomRight topRight center focus|
}}
  <p>Top left corner is: {{topLeft}}</p>
{{/with}}
```

The problem with this code is that it's difficult to read and edit. In order to understand which which value belongs to which variable, you have to count both names and values.

The common workaround is to pass a hash instead:

```handlebars
{{#with (hash
  topLeft     = (some-math a b c)
  topRight    = (some-math b a c)
  bottomLeft  = (some-other-math b d)
  bottomRight = (some-math c a b)
  center      = (some-other-math d d)
  focus       = (some-other-math d a b)
  as |locals|
)}}
  <p>Top left corner is: {{locals.topLeft}}</p>
{{/with}}
```

Much better, but now we have to reference every variable via the `locals.` prefix which is tedious. Some developers prefer to reduce it to one character, e. g. `l.`, which does not contribute to code clarity.

And when you have nested `with`s, you have to come up with unique hash names. "Was this property I need in `locals2` or `locals3`? I'll have to scroll up to find out."

That said, **passing a hash to `with` is such a common trick in the Ember that we should consider making it official: allow passing named arguments to the `with` helper while **removing all the crust**.

This RFC is also intended as a lightweight alternative to [`let` it be: bind names to values in templates](https://github.com/emberjs/rfcs/pull/200) that stays true to the Ember way and doesn't violate the declarative nature of templates.


# Detailed design

The `with` helper should accept named (non-positional, hash-like arguments).

In order to avoid ambiguity, the compiler should forbid mixing positional and named arguments together.

When at least one named argument is provided, the `as |foo|` part should be left out.

Here's the initial example again:


```handlebars
{{#with
  topLeft     = (some-math a b c)
  topRight    = (some-math b a c)
  bottomLeft  = (some-other-math b d)
  bottomRight = (some-math c a b)
  center      = (some-other-math d d)
  focus       = (some-other-math d a b)
  foo         = "Bar"
}}
  <p>Top left corner is: {{topLeft}}</p>
{{/with}}
```


# How We Teach This

The terminology is already established: named arguments (named parameters). We can also refer to this technique as "hash-like syntax".

Publishing a blog post and updating the guides should be pretty straightforward. This proposal doesn't change anything in Ember, just makes a small addition to the Handlebars syntax.


# Drawbacks

The only caveat is that variables can now be introduced into the template scope without the `as |foo|` construct.

This shouldn't be a big deal as the whole point of the `with` block helper is to introduce new variables, thus code understandability does not suffer.


# Alternatives

### 1. Positional arguments

```handlebars
{{#with
  (some-math a b c)
  (some-math b a c)
  (some-other-math b d)
  (some-math c a b)
  (some-other-math d d)
  (some-other-math d a b)
  as |topLeft bottomLeft bottomRight topRight center focus|
}}
  <p>Top left corner is: {{topLeft}}</p>
{{/with}}
```

Bulky, difficult to read and update as it forces you to count both names and values, and is prone to introducing errors when updating.

### 2. Nested `with`s

```handlebars
{{#with (some-math a b c) as |topLeft|}}
  {{#with (some-math b a c) as |bottomLeft|}}
    {{#with (some-other-math b d) as |bottomRight|}}
      {{#with (some-math c a b) as |topRight|}}
        {{#with (some-other-math d d) as |center|}}
          {{#with (some-other-math d a b) as |focus|}}
            <p>Top left corner is: {{topLeft}}</p>
          {{/with}}
        {{/with}}
      {{/with}}
    {{/with}}
  {{/with}}
{{/with}}
```

This makes variable names and values appear together, but at ridiculous price.


### 3. RFC #200 [`let` it be: bind names to values in templates](https://github.com/emberjs/rfcs/pull/200)

That RFC proposal raised concern in discussion.
