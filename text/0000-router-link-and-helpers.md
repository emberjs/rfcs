- Start Date: 2018-06-15
- RFC PR:
- Ember Issue:

# Add a Link component and routing helpers

## Summary

The purpose of this is to add a single component, `Link`, that wraps up some routing helpers for default usage of linking / routing.

## Motivation

Positional arguments are not planned for use with Angle Bracket invocations. For clarity, and with some inspiration from react-router, It may be beneficial for Ember to include a `Link` component, similar to the existing `link-to`, but with only named arguments. This will help with clarity of intent, and also allow for easier transitioning from other communities who already have named-argument links.

## Detailed design

*Phase 1*
- add built-in helpers for custom link-like behavior
- add `Link` component with `@to` named argument -- all other named arguments will be the params

*Phase 2*
- deprecate `LinkTo`

*Phase 3*
- remove `LinkTo` and the corresponding deprecation

### Details

*Helpers to be implemented*

 - `transition`
    this is a helper that would tie into the router to be able to achieve the same functionality as the navigational usage of `link-to`

    examples:

      ```hbs
      <button {{action (transition to='posts.edit' postId=post.id)}} />
        Edit
      </button>

      <button {{action (transition to='login' replace=true)}} />
        Login
      </button>
      ```



 - `is-route-active`
    this returns a boolean representing whether or not the current route matches the passed argument.

    example:

      ```hbs
      <button
        {{action (transition to='posts.edit' post=model.post)}}
        class={{if (is-route-active 'posts.edit' post=model.post) 'selected'}} />
      ```

   this may need to support multiple invocation styles, for when there aren't parameters and the dev wants to be concise:

   examples:

     ```hbs
     <button ... class={{if (is-route-active 'posts') 'selected'}} />
      No model params provided
     </button>
     ```


*Add `Link` Component*

`Link` could just be tagless component with the following template:

```hbs
{{!-- components/link/template.hbs --}}
<a
  {{action (transition @to models=@models replace=@replace)}}
  ...attributes
  class="{{if (is-route-active @to models=@models) 'active'}}">

  {{yield}}

</a>
```

Usage:
```hbs
<Link @to='posts.edit' @models={{post}}>Edit Post</Link>
```

where `@models` can be a `hash`, `array`, or just a single object or id.

Starting with @rwjblue's [ember-router-helpers](https://github.com/rwjblue/ember-router-helpers) as a base, `is-route-active` is already implemented (as `is-active`, so this would need to be renamed), along with `route-params`, `transition-to`, and `url-for`.

A [PR](https://github.com/rwjblue/ember-router-helpers/pull/46) has been started that upgrades `ember-router-helpers` and adds tests in preparation for implementing the helper renames and `Link` component.



**Deprecation** `link-to`

The goal of `Link` and the route helpers is to provide a flexible way of routing from the template while providing a sample component with sensible defaults. This would achieve the exact same functionality as `link-to`, so `link-to` would no longer be needed and could eventually be removed.

## How we teach this

We'll want to make it very clear that the traditional `link-to` technique of linking will still be available, but will eventually be deprecated.

The documentation and guides would need to be incrementally upgraded to inform people about the new linking strategy -- but because the old way would still work, having some docs say `link-to` instead of `Link` wouldn't be the worst thing.

## Drawbacks

The biggest drawback is that all routing documentation would be out of date, and there is a lot of routing documentation and blog posts throughout the web.

Without positional params, the alternative may need to use the `array` helper for the `@to` argument... which would feel awkward enough to make people wonder why `link-to` (as `LinkTo`) didn't get a named argument for the route / route params. With the `Link` component, the arguments would read more ergonomically -- `<Link @to=...` vs `<LinkTo @route=...`. The latter implies that things other than routes could be linked to via the `link-to` component, which could cause confusion about the intended usage.

## Alternatives

1. Only implement the routing helpers
  - people could define their own linking components however they desire

## Inspiration / Code taken from
- https://github.com/alexspeller/ember-cli-active-link-wrapper
- https://github.com/BBVAEngineering/ember-route-helpers
- https://github.com/peec/ember-transition-helper
- https://github.com/rwjblue/ember-router-helpers
