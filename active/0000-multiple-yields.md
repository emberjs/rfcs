- Start Date: 2015-04-01
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Allow templates to have multiple `{{yield}}`'s.

# Motivation

Today components and layouts are limited to a single `{{yield}}` which makes it difficult to deal with even slightly complex layouts and components.

# Detailed design

```hbs
{{!-- templates/components/my-component --}}
<div class="my-component">
  <div class="my-component-body">
    {{yield}}
  </div>

  {{#has-content-for "footer"}}
    <div class="my-component-footer">
      {{yield content-for="footer"}}
    </div>
  {{/has-content-for}}
</div>
```

```hbs
{{!-- templates/index.hbs --}}
{{#my-component}}
  <p>Here is some body text</p>

  {{#content-for "footer"}}
    <p>Here is some footer text</p>
  {{/content-for}}
{{/my-component}}
```

```html
<!-- Rendered Output -->
<div class="my-component">
  <div class="my-component-body">
    <p>Here is some body text</p>
  </div>

  <div class="my-component-footer">
    <p>Here is some footer text</p>
  </div>
</div>
```

# Drawbacks

N/A

# Alternatives

@stefanpenner mentioned some ideas he had, I'll let him explain them.

# Unresolved questions

N/A
