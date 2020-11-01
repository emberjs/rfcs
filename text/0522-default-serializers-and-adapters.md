---
Start Date: 2019-07-27
Relevant Team(s): Ember Data
RFC PR: https://github.com/emberjs/rfcs/pull/522
Tracking: 

---

# Deprecate default Adapter and Serializer fallbacks

## Summary
As part of Project Trim, https://github.com/emberjs/data/issues/6166, this deprecates the fallback and default adapter and serializer across ember-data including:
1. deprecate -default serializer fallback in Store.serializerFor
2. deprecate `adapter.serializer` and `adapter.defaultSerializer` instance property fallbacks (which currently default to -json-api).
3. deprecate `store.defaultAdapter` instance property (which defaults to `-json-api`) and the `-json-api` adapter fallback behavior in `adapterFor`.
4. deprecate `record.toJSON` instance method since this relies on the `-json` serializer.

## Motivation

The adapter and serializer packages provide reference implementations and base classes that are not required for applications that implement their own following the required interfaces for adapters and serializers as defined in their respective base classes.  Deprecating them allows us to simplify the lookup pattern and remove automatic injection and registration of potentially unused classes.

In addition to removing use of initializer injection, this takes a significant step toward simplifying the mental model for how to determine what adapter/serializer is in use. Removing the defaults forces app developers to be more cognizant about the type of application level concerns vs model-specific concerns; they will now need to explicitly define and use specific adapters/serializers. After this deprecation RFC lands, apps will always use an adapter/serializer explicitly put into your application and the rule will be "the adapter matching modelName falling back to application".

## Detailed design
The injection of `-default` and `-json-api` serializers will be removed in the next major version (4.0). Since this changes some core assumptions we will deprecate the reliance on the existence of the defaults. All deprecation warnings will only be shown in Dev mode.

##### deprecate -default serializer fallback in store.serializerFor
A deprecation warning will be shown when [no model, application or adapter serializer](https://github.com/emberjs/data/blob/67affb0eca7048a1a0edc856af46d1305cd1fc1d/packages/store/addon/-private/system/store.ts#L2909) has specified and the default must be used. We will recommend implementing an application serializer.

##### deprecate adapter.serializer and adapter.defaultSerializer fallbacks
A deprecation warning will be shown when accessing the [adapter's defaultSerializer](https://github.com/emberjs/data/blob/67affb0eca7048a1a0edc856af46d1305cd1fc1d/packages/store/addon/-private/system/store.ts#L2896). This will be distinct from the warning about using the application level default. We will recommend implementing an application serializer.

##### deprecate store.defaultAdapter (-json-api) and the -json-api adapter fallback behavior
A deprecation warning will be shown when the [defaultAdapter](https://github.com/emberjs/data/blob/b0cf3225662bfb806cd0c02b55b763e37a319b32/packages/store/addon/-private/system/store.ts#L309) is accessed.  We will recommend implementing an application adapter.

##### deprecate record.toJSON
A deprecation warning will be shown when toJSON is called since it uses a serializer to create a JSON representation of [the model](https://guides.emberjs.com/release/models/customizing-serializers/#toc_customizing-serializers). Users may call record.serialize() or implement their own toJSON instead.

## How we teach this

Today we have extensive documentation about creating custom serializers, but we will need to update the guides to specify the desired serializer in [app/serializers/application.js](https://guides.emberjs.com/release/models/customizing-serializers/#toc_customizing-serializers)

The deprecation guide app will be updated with examples showing how to
migrate away from relying on the defaults.

## Drawbacks

The drawback to making this change is that apps relying on the default serializer need to add some boilerplate to explicitly set the serializer.

## Alternatives

We could not do this.
