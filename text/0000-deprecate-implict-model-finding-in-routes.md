- Start Date: 2019-11-27
- Relevant Team(s): Ember.js, Ember Data, Learning
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Deprecate Implicit Record Finding In Routes

## Summary

Today, Ember's Route's `model` hook has an implicit convention based on the name and casing of a dynamic segment / parameter passed to the model hook. 
For example, if a dynamic segment is `:post_id`, there exists logic to split on the underscore, look up a service on the route named `store`, and find a record of type `post`.  This is far from the explicit and discoverable data loading path we teach today and deprecating this code for removal will help align with our current easier-to-teach/explict data loading patterns.

## Motivation

Recently, [eslint-plugin-ember](https://github.com/ember-cli/eslint-plugin-ember/issues/410) introduced a lint rule to _enforce_ the `snake_case` convention of dynamic segments. I and a few others don't like `snake_case` in JavaScript, as the convention is `camelCase`. We looked into submitting and RFC for changing the default to `camelCase`, but encountered [this code](https://github.com/emberjs/ember.js/blob/b4456779d0f5f5b99da853c4e1f0688ce96cc27c/packages/%40ember/-internals/routing/lib/system/route.ts#L1167-L1186) that suggests some implicit data loading behavior for `snake_case` params _ending_ in `_id` and if you _happen_ to have a service named `store`.

## Transition Path

We don't teach this behavior, so the transition path should be fairly light.

if someone _doesn't_ define a `model` hook on their route, and are taking advantage of the implicit record loading behavior, they will be presented with a deprecation warning in the console [triggered from here](https://github.com/emberjs/ember.js/blob/b4456779d0f5f5b99da853c4e1f0688ce96cc27c/packages/%40ember/-internals/routing/lib/system/route.ts#L1209) -- showing that removal will happen in Ember v4.

A person currently using this implicit behavior should add to their route:

```ts
async model({ post_id }) {
  return this.store.find('post', post_id);
}
```

and if a route doesn't exist in the app, it will need to be defined.

## How We Teach This

Ember Guides shouldn't need updating.

Adding an entry to the deprecation guides should be sufficient, briefly explaining the before/after from above should be sufficient.

## Drawbacks

Drawbacks only exist for people who are utilizing this implicit behavior. For them, the drawbacks are:
 - they now have to define a model hook
 - they may have to create a route file where previously the route was implicitly defined for them and implicitly loading the record for them as well

## Alternatives

Impact of not doing this: stale code, added weight for everyone else who isn't using this feature, or who isn't using ember-data.

## Unresolved questions

None at this time.