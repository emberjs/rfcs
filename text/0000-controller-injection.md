- Start Date: 2020-01-09
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/574
- Tracking:

# Deprecate Controller Injection

## Summary

This RFC recommends deprecating the ability to inject a controller into another.

```js
import { inject as controller } from '@ember/controller';

export default SomeController extends Controller {
    @controller other;

    get otherFoo() {
        return this.other.foo;
    }
}

// app/controllers/other.js
export default OtherController extends Controller {
    foo = 1;
}

///////
/////// In EmberObject.extend parlance:
///////
import { inject as controller } from '@ember/controller';

export default Controller.extends({
    other: controller(),

    otherFoo: computed('other.foo', function() {
        return this.other.foo;
    })
})

// app/controllers/other.js
export default Controller.extend({
    foo: 1
})
```

## Motivation

Controller injection is not a standard paradigm in Ember. Data that needs to be shared between
routes or controllers or components can be shared via services. Alternatively, if a controller
needs to be accessed, the `controllerFor()` API can be used or, as a last resort, the `lookup`
API on the ApplicationInstance (i.e. the `owner` of most Ember core class instances) can be used.

Deprecating this functionality also makes it ever-so-slightly easier to make the case for removing
controllers entirely from Ember.

## Detailed design

The `inject` function from the `@ember/controller` package should log a deprecation warning and
then eventually stop being exported.

## How we teach this

There is not much to teach for this removal as this is not a major feature and it is not mentioned
in the guides. There are other efforts to teach Ember developers about how to share state around
the application.

## Drawbacks

It may be seen as unnecessary churn if controller injection is removed as a feature without a longer
term plan for controllers.

## Alternatives

- Keep controller injection.
- Wait for the long term plan for controllers before making any changes.
    - I think it's good to start deprecating / removing features to narrow the scope of controllers.

## Unresolved questions

- What code needs to be removed once the inject function is removed?
