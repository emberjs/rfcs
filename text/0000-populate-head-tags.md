- Start Date: 2019-06-21
- Relevant Team(s): Ember.js, Learning
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Support Populating head tags in Routes

## Summary

Currently, an addon is used for manipulating head data, [ember-cli-head](https://github.com/ronco/ember-cli-head),
but it would be nice to support adding head tags through a built in hook in Ember routes.

## Motivation

Setting head data and meta values is very important for modern web apps, and helps with SEO, 
link unfurling, social media, etc. With a dedicated `head()` hook, we could bake first class support
for setting these values into Ember itself. 

## Detailed design

The format for the `head()` hook would mimic the one used in [Nuxt.js](https://nuxtjs.org/api/pages-head#the-head-method).

It would be something like:

```js
import Route from '@ember/routing/route';

export default Route.extend({
  head () {
    return {
      title: this.title,
      meta: [
        { hid: 'description', name: 'description', content: 'My custom description' }
      ]
    }
  }
});
```

We would then have to loop through the `meta` array and set `meta` tags with the properties listed in each
object. 

i.e.

```html
<meta name="description" content="My custom description">
```

I am not sure on the implementation details, but if there is interest in something like this,
I will investigate it further.

## How we teach this

This would likely need its own section in the guides, but should not need a ton of
explanation. Just an example explaining the hook name, and the format of the data
expected to be returned from it.

## Drawbacks

Implementing this internally could add a very small amount of bloat to Ember apps
that do not need this feature.

## Alternatives

We could keep this as a separate addon, but perhaps try to make it a more official,
core team endorsed addon.
