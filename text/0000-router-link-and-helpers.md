- Start Date: 2018-06-15
- RFC PR:
- Ember Issue:

# Add a Link component and routing helpers

## Summary

The purpose of this is to add a single component, `Link`, that wraps up some routing helpers for default usage of linking / routing.

## Motivation

Currently, on ember#canary, the Angle Bracket Invocation feature doesn't support positional arguments. For clarity, and with some inspiration from react-router, It may be beneficial for Ember to include a `Link` component, similar to the existing `LinkTo` (`link-to`), but with only named arguments. This will help with clarity of intent, and also allow for easier transitioning from other communities who already have named-argument links.

## Detailed design

#### tl;dr:

*Phase 1*
- add named arguments to `LinkTo`: `@routeName` and `@models`
- add built-in helpers for custom link-like behavior

*Phase 2*
- add `Link` component with `@to` named argument -- all other named arguments will be the params
- deprecate `LinkTo`

*Phase 3*
- remove `LinkTo` and the corresponding deprecation


#### Phase 1
*Add Named Arguments to `LinkTo`*

  `@routeName` and `@models` will allow us to smoothly transition away from `{{link-to}}` into the `<AngleBracket />` invocation-style.

  - example:

  ```hbs
    {{!-- before --}}
    {{#link-to 'posts.edit' post.id}}
      Edit
    {{/link-to}}

    {{!-- After --}}
    <LinkTo @routeName='posts.edit' @models={{hash postId=post.id}}>
      Edit
    </LinkTo>
  ```


  The `<AngleBracket />` invocation with named arguments is more verbose, but trade-off here is the added clarity of what argument is used for what purpose -- especially with respect to the models / model ids.

*Helpers to be implemented*

 - `transition`
    this is a helper that would tie into the router to be able to achieve the same functionality as the navigational usage of `LinkTo`

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

#### Phase 2

*Add `Link` Component*

`Link` could just be tagless component with the following template:

```hbs
{{!-- components/link/template.hbs --}}
<a
  {{action (transition @to models=@models)}}
  class="{{if (is-route-active @to models=@models) 'active'}}">

  {{yield}}

</a>
```

Usage:
```hbs
<Link @to='posts.edit' @models={{post}}>Edit Post</Link>
```

where `@models` can be a `hash`, `array`, or just a single object or id.




**Deprecation** `LinkTo` aka `link-to`

The goal of `Link` and the route helpers is to provide a flexible way of routing from the template while providing a sample component with sensible defaults. This would achieve the exact same functionality as `LinkTo`, so `LinkTo` would no longer be needed and could eventually be removed.

It's possible we could write a codemod to auto-convert everyone's non-angle-bracket invocation of `{{#link-to ...` to the new angle bracket component: `Link`

## How we teach this

We'll want to make it very clear that the traditional `LinkTo` technique of linking will still be available, but will eventually be deprecated.

The documentation and guides would need to be incrementally upgraded to inform people about the new linking strategy -- but becaus the old way would still work, having some docs say `LinkTo` instead of `Link` wouldn't be the worst thing.


## Drawbacks

The biggest drawback is that all routing documentation would be out of date, and there is a lot of routing documentation and blog posts throughout the web.

Without positional params, the alternative may need to use the `array` helper for the `@to` argument... which could almost make people wonder why `LinkTo` didn't get a named argument for the route / route params. With the `Link` component, the arguments would read more ergonomically -- `<Link @to=...` vs `<LinkTo @route=...`. The latter implies that things other than routes could be linked to via the `LinkTo` component, which could cause confusion about the intended usage.

## Alternatives

There are two alternatives:

1. Do nothing:
  - people will find that the `LinkTo` usage with angle bracket invocation to be somewhat awkward, maybe unintuitive.
2. Only implement the routing helpers
  - people could define their own linking components however they desire

## Unresolved questions

- Should we offer additional click APIs on `Link`? such as:
  - onclick / onclick
  - beforeTransition
  - etc?

- Would it make sense to somehow have a way to statically confirm that the route path is valid? right now, it looks like a string -- how would I know if I typed it wrong?

## Inspiration / Code taken from
- https://github.com/alexspeller/ember-cli-active-link-wrapper
- https://github.com/BBVAEngineering/ember-route-helpers
- https://github.com/peec/ember-transition-helper
- https://github.com/rwjblue/ember-router-helpers
