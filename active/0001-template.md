- Start Date: 2014-08-29
- RFC PR: 
- Ember Issue: 

# Summary

## Ember Mixer

The idea is to be able to Mixin various orthogonal functionality (much like Aspects) into places where they belong so as to adhere to Single Responsibility. This could just as well be a plugin/addon called ember-mixer for now? 

# Motivation

Why are we doing this?  What use cases does it support? What is the expected outcome?

Makes the app more easily configurable as to where to insert common aspects such as security strategies, transaction logic, logging etc.
Better to define this declaratively in a central place instead of having to maintain this all over the app. Becomes a maintenance "nightmare" for large apps!?

## Examples

```javascript
Ember.Mixer.onAll('controllers').mixin('authorizer');
Ember.Mixer.onAll('controllers').matching([/^Admin/]).mixin(['authorizer']);
```

```javascript
export mixinRecipes = {
  '@admins': ['^Admin', 'Admin$']

  controllers: {
    '@admins': ['^Admin', '*AdminController'] // overides global alias!

    authorize: {
      matches: ['^Admin'], mixin: ['authorizer']
    },
    log: {
      // custom matcher and mixin functions
      matches: function(mod){}, mixin: function(target, mod) {}
    }      

  },
  authenticate: {
    types: ['controllers', routes], matching: '@admins', mixin: ['authentication']
  }      
}

// mixinRecipes - main API method
// should take a mixin recipe and convert into executions like this:
// Ember.Mixer.onAll('controllers').mixin('authorizer');
// Ember.Mixer.onAll('controllers').matching([/^Admin/]).mixin(['authorizer']);

Ember.Mixer.mixinRecipes(mixinRecipes);
```

# Detailed design

Something like this (pseudo):

```javascript
// utility code ("pseudo")

Ember.Mixer = Ember.Object.extend(Ember.MixFinder, {
  onAll: function(type) {
    return Ember.MixMatcher.extend({
      modules: this.findAll(type),
      type: type
    })    
  }
})


Ember.MixFinder = Ember.Mixin.create({
  findAll: function(type) {
    Ember.__container__.lookup('application:' + type);
  },
  toArr: function(args) {
    return [].slice.call(args, 0);
  }
})

Ember.MixinMixer = Ember.Object.extend(Ember.MixFinder, {
  type: null,
  mixin: function (modules) {
    var allMixins = this.findAll('mixins');
    
    this.toArr(modules).forEach(function(mod) {      
      allMixins.get(name).apply(mod)
    })  
  }
})

// 
// allows chaining with .matching(...)
// also mixes in MixinMixer which provides .mixin(...)
Ember.MixMatcher = Ember.Object.extend(Ember.MixinMixer, {  
  // API
  // mixin
  matching: function (exprs) {
    return Ember.Mixer({
      modules: this.matchedModules(this.toArr(exprs)),
      type: this.type
    });
  }

  // private from here
  modules: [],

  // Finds modules matching one or more expressions on name
  matchedModules: function(matchExprs) {
    var self = this;
    return matchExprs.map(function match(expr) {
      return self.modules.select(function(module) {
        return module.match(expr); 
      })      
    });
  }
}
```

# Drawbacks

This is quite a big change and perhaps the whole MetaMagic part should reside in a separate (addin /plugin) to be included as an opt-in.
The basic Namespace part first described could however be part of core.

The MetaMagic part just goes to demonstrate some of the power this sort of approach would allow ;)

# Alternatives

What other designs have been considered? What is the impact of not doing this?

See drawbacks

# Unresolved questions

What parts of the design are still TBD?

You tell me...