- Start Date: 2019-03-10
- Relevant Team(s): Ember CLI
- RFC PR: [#462](https://github.com/emberjs/rfcs/pull/462)
- Tracking: (leave this empty)

# Configuring addon modules in Module Unification layout

## Summary

Provide an API for addons to configure their module types and collections for Module Unification apps.

## Motivation

The Module Unification layout is described on the [RFC-143](https://github.com/emberjs/rfcs/blob/master/text/0143-module-unification.md). At this moment Classic addons that require apps to locate classes into a non-default app folder do not work "out of the box" in an Module Unification application.

For example, the ember-simple-auth addon allows defining the application authenticators on the `authenticators` folder.

> The authenticator to use is chosen when authentication is triggered via the name it is registered with in the Ember container:

```js
this.get('session').authenticate('authenticator:custom')
```

App developers only need to locate the module on the **expected project folder** to make the addon work.

```js
// app/authenticators/custom.js
import Base from 'ember-simple-auth/authenticators/base';

export default Base.extend({
  ...
});
```

In an Module Unification application, you would likely locate `custom`  within `src/authenticators` but this will not work **automatically**.

The Module Unification RFC provides a broader convention for naming and organizing a project's modules, and the options of the application module structure must be set up in the application resolver for MU applications. You can read more about the [MU Module Naming and Convention](https://github.com/emberjs/rfcs/blob/master/text/0143-module-unification.md#module-naming-and-organization).

In Ember, the application resolver defines the lookup rules to resolve container lookups. The Resolver class has been updated to support the MU layout, and the blueprint will generate the following `src/resolver.js`:

```js
import Resolver from 'ember-resolver/resolvers/fallback';
import buildResolverConfig from 'ember-resolver/ember-config';
import config from '../config/environment';

let moduleConfig = buildResolverConfig(config.modulePrefix);

export default Resolver.extend({
  config: moduleConfig
});
```

The config option on the MU resolver is a hash that includes the `types` and `collections` mappings defining the rules to resolve container lookups. Now, an app can update the default resolver configuration to add their types and collections like:

```js
moduleConfig.collections = Object.assign(moduleConfig.collections, {
  // ember-simple-auth
  authenticators: {
    types: ['authenticator'],
    defaultType: 'authenticator'
  }
});
```

Because there is not an API for addons to add their module types or collections, developers have to add them in the application resolver to make the addon works in an MU application.
The following proposal provides an API for addon authors to add their types and collections in order application developers do not have to configure them.


## Detailed design

The API proposal provides a new hook `resolverConfig` for addons to define their module types and collections.

```js
// my-addon/index.js
module.exports = {
  name: require('./package').name,
  resolverConfig() {
    return {
      collections: {session: { definitiveCollection: 'session' }},
      types: {translation: { definitiveCollection: 'main' }}
    };
  }
};
```

The types and collections for all the application addons will be automatically merged into the default MU types and collections.

The app will throw an exception if:

- an addon tries to overwrite a type or collection mapping that is included on the default MU configuration.

- two addons define the same type or collection.

For more implementation details, there is an [open PR](https://github.com/ember-cli/ember-resolver/pull/348).

## How we teach this

The guides should include a section explaining how to extend the default resolver configuration for application developers or addon authors.

The "writing addons" tutorial has to document the `resolverConfig` hook.

## Drawbacks

The proposal does not allow:

- an addon to extend the Module Unification types and collections.

- two addons to register the same type or collection.

This behaviour can be modified.

## Alternatives

1) Don’t change anything and app developers need to register their addon types and collections based on the addon instructions.

2) Provide a different API than the `resolverConfig` hook.

Even though it is related, this RFC does not try to define the API to define Module Type and Module Collections on the resolver configuration; other RFC should cover any alternative to this API.

## Unresolved questions

The Drawbacks section comments the unresolved questions.
