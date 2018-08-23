* Start Date: 08/22/2018
* RFC PR: (leave this empty)
* Ember Issue: (leave this empty)

# Return a Promise from DS.Model.save()

## Summary

DS.Model.save() will return an RSVP.Promise instead of a PromiseObject.

## Motivation

* The API documentation already documents the return value as a Promise.
* Remove dependency on promise proxies
* Async Consistency - The PromiseObject encourages a usage pattern of sometimes-async, sometimes-sync behavior. We want to [reduce zalgo](http://blog.izs.me/post/59142742143/designing-apis-for-asynchrony).
* API consistency - While the PromiseObject proxies properties, the PromiseObject does not proxy methods on the underlying model. For instance, if you try to call a method like destroyRecord on the model, you will get a “not a function” error because the method is called on the Proxy and not the underlying model. This encourages users for Ember Data to reach into private implementation details of the promise proxy, such as using proxy.get('content') to call methods.
* Enable new functionality in Ember Data while making backwards compatibility with older versions of Ember easier.
* The documentation of using PromiseProxy APIs has been replaced over time in the guides and API documentation to use more typical JavaScript usage of Promises, such as async/await, instead of PromiseProxies.

## Detailed design

Introduce a new feature flag, “ds-return-promise-from-model-save”. When enabled, DS.Model.save() will return an RSVP.Promise that resolves with the model if the save was successful, or rejects with an error if the save fails. When disabled, a PromiseObject will be returned to keep today's behavior, but accessing properties on the PromiseObject or calling non-Promise functions (.then, .finally, .catch) on the PromiseObject will issue a deprecation warning.

Deprecation Plan:

The deprecations can be implemented by adding deprecations around the existing PromiseObject. However, we can't include the deprecations in the actual PromiseObject class because [it is the base class for the relationship proxies](https://github.com/emberjs/data/blob/master/addon/-legacy-private/system/promise-proxies.js#L85). Instead, we can create and use a new DeprecatedPromiseObject for the DS.Model.save() function. It functions like a PromiseObject but deprecates the following behavior:

    * Accessing properties on the PromiseObject returned from DS.Model.save()
        * Use deprecateProperty from Ember to deprecate all the properties that the PromiseProxy Mixin provides. The list of properties can be found here: https://github.com/emberjs/ember.js/blob/master/packages/ember-runtime/lib/mixins/promise_proxy.js
        * Deprecate accessing any unknown properties. For instance, someone might try to grab the value of an attr or computed property from a model, e.g. proxy.get('email').
    * Calling functions that are not available on RSVP.Promises
        * Not Safe (issue deprecation): .get/.set/anything else really
        * Safe (do not issue a deprecation): .then, .catch, .finally

The implementation follows the standard feature-flagging process in Ember and Ember Data. The final implementation will look something like:

```
  save(options) {
    const savePromise = this._internalModel.save(options).then(() => this);
    
    if (flagEnabled("ds-return-promise-from-model-save")) {
      return savePromise;
    } else {
      return DeprecatedPromiseObject.create({
        promise: savePromise
      });
    }
  },
```

## How we teach this

We do not believe this requires any update to the Ember curriculum. API documentation does not need to be updated as it is already documented as returning a promise. This behavior of accessing properties through the PromiseProxyMixin specifically for DS.Model.save() is not documented in the guides.

## Drawbacks

For users relying on this behavior, they will have to refactor their code to either use patterns like async/await, ember-concurrency, or ember-promise-helpers. Alternatively, if all they want to access are properties on the model, they can use the model instead.

## Alternatives

The impact of not doing this prevents further changes in Ember Data and requires more effort to maintain compatibility between Ember Data and older versions of Ember.

## Unresolved questions

* It's quite possible Ember Data users use the isSettled, isRejected, isFulfilled, and isPending properties of the  PromiseProxyMixin to display state in their UI. For example, you may have a component that calls record.save(), and uses isPending to show a loading spinner or isRejected to show UI for error states. Should we point them to community alternatives like [ember-promise-helpers](https://github.com/fivetanley/ember-promise-helpers) or [ember-concurrency](http://ember-concurrency.com/docs/introduction/) in the deprecation message for these properties?
