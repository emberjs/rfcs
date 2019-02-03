- Start Date: 2019-02-01
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking:

# Route Model Name Property

## Summary

Add a property, `modelName`, to the route that will be the key that the model hook's return value is assigned to in the controller.

## Motivation

The default key for the routes model hook return on the controller is `model`. This name can be vague. The name is only expressive in the case where the route/controller naming represents the resource, e.g. `foo` route returns a `foo` model instance, and relies on the developer being aware of the convention, and for no other developer to ignore (circumvent?) the convention and return something else in the model hook. By adding a `modelName` property we allow developer to be expressive, and it will help future developers understand explicitly in the controller what type of model they are working with without having to refer back to the model hook.

Additionally it adds some context to the current behaviour of assigning the model hook return to the controller, which is not transparent

## Detailed design

- Add a property called `modelName` to the Route API. When the controller is initialised, the return value from the routes `model()` hook is assigned to the controller with value of the `modelName` property as the key.
- This change would not require us to stop assigning the return value to `model` key on the controller as well.
- Generator could be updated so that `ember generate route foo` would add `modelName: foo` to the generated route file.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

This doesn't introduce any new concepts or patterns to Ember, however this API is more explicit and user friendly than the current behaviour. At the moment the fact that the return from the `model()` hook on the route will be assigned to the controllers `model` key is _ember magic™️_, which must be taught.

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

We would continue to mention the application of the model hook in the guides as we currently do:

> [_"The route will then set the return value from the model hook as the model property of the controller."_](https://guides.emberjs.com/release/routing/specifying-a-routes-model/)

We could add an extra sentence covering this feature:

You can also set the return value from the model hook to be any custom property name on the controller by using `modelName` property in the route.
```
app/routes/posts.js
import Route from '@ember/routing/route';

export default Route.extend({
  model() {
    return this.store.findAll('post');
  },
  modelName: articles
});
```
```
app/templates/posts.hbs
<h1>Articles</h1>
{{#each this.articles as |article|}}
  <p>{{articles.body}}</p>
{{/each}}
```
> How should this feature be introduced and taught to existing Ember
users?

We can let developers know that instead of doing:
```
app/routes/posts.js
import Route from '@ember/routing/route';

export default Route.extend({
  model() {
    return this.store.findAll('post');
  },
  setupController(model, controller){
    controller.set('customModelName', model);
  }
});
```
They can simply use  the `modelName` property on the route, if they want to 😀.

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

Increases the size of the API.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

I thought about using decorators, but I'm unsure if that would even be feasible.

I opened this PR to get feedback and hear other alternatives.

I don't consider this to be essential feature, the RFC is born out of my personal frustration with Ember. In large code bases I often find that the return from the model hook is not the resource represented by the file name. I make the assumption that the `model` in the controller is what the convention suggests it would be, and it's a reoccuring 'gotcha' moment, that costs me time when I have to debug and realise that the `model` is different to the expectation I have from Ember's conventions.


## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
