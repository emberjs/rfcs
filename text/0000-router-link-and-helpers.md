- Start Date: 2018-06-15
- RFC PR: 
- Ember Issue: 

# Add a Link component and routing helpers

## Summary

The purpose of this is to add a single component, `Link`, that wraps up some routing helpers for default usage of linking / routing.

## Motivation

Currently, on ember#canary, the Angle Bracket Invocation feature doesn't support positional arguments. For clarity, and with some inspiration from react-router, It may be beneficial for Ember to include a `Link` component, similar to the existing `LinkTo` (`link-to`), but with only named arguments. This will help with clarity of intent, and also allow for easier transitioning from other communities who already have named-argument links.

## Detailed design

Helpers to be implemented:
 - `transition` 
    this is a helper that would tie into the router to be able to achive the same functionality as the simple usage of `LinkTo`

    example:
      
      ```
      <button {{action (transition to='posts.edit' postId=post.id}}
      ```
    
 - `replace-with`
    this is same as transition, but replaces the current URL and current history entry. This may be an argument to transition instead. e.g.: `(transition ... replace=true)`
    
 - `is-route-active`
    this returns a boolean representing whether or not the current route matches the passed argument.
    
    example:
    
      ```
      <button 
        {{action (transition to='posts.edit' post=model.post)}} 
        class={{if (is-route-active name='posts.edit' post=model.post) 'selected'}} />
      ```
      
   this may need to support multiple invocation styles, for when there aren't parameters and the dev wants to be concise:

   example:
   
     ```
     <button ... class={{if (is-route-active 'posts') 'selected'}} />
     ```

Inspiration for helpers implementation could come from https://github.com/BBVAEngineering/ember-route-helpers

 Components to be implemented:
 - `Link`
 
`Link` could just be tagless component with the following template:
```hbs
{{!-- components/link/template.hbs --}}
<a 
  {{action (transition ...@to)}} 
  class="{{if (is-route-active ...@to) 'active'}}"
>

{{yield}}

</a>
 ```

Usage:
```hbs
<Link @to=(array 'posts.edit' post)>Edit Post</Link>
```


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

- Is there a way for making `(array 'posts.edit' post)` more ergonomic? _having_ to use the array helper _seems_ like it shouldn't be a thing. Maybe if the `transition-to` helper took named arguments without `hash`?  transition can be implemented to support this:`(transition to='posts.edit' postId=post.id)`.. but for `Link`, I don't know how the route arguments would get passed to `transition`.

- For routes with multiple segment parameters, it may become hard to keep track of things. e.g.: `(array 'posts.edit.comments.reply' post comment.id)` how do I know where the parameters are going to go without looking at the router.js file? (though, this is an existing question)

- Would it make sense to somehow have a way to statically confirm that the route path is valid? right now, it looks like a string -- how would I know if I typed it wrong?
