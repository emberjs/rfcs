- Start Date: 2014-9-22
- RFC PR: 
- Ember Issue: 

# Summary

Here in this RFC I'd like to propose a new container built to work with ES6 modules and the ES6 module loader. The new system doesn't try to hide the module system, it just provides dependency injection.

# Motivation

Ember's current container really works very well as a dependency injection system. Lately, however, many users have reached the limits of the current system:
- Namespaces: The current way to look something up (e.g. "view:application") has the same drawbacks as the global scope in JavaScript. If two 3rd party libraries register an item with the same name, a name collision occurs.
- Lazy loading: In large applications it becomes necessary to selectively load just the parts that are needed for the current route.

The main reason for this RFC is that I'd like to start a discussion on this topic, because I think it's important. The presented solution isn't complete, but it's a start and I'd like to have some input.

# Detailed design

## Work in conjunction with ES6 module loader

Here's how it looks like to load a module using the ES6 module loader:

``` JavaScript
System.import('my-module')
  .then(function(module) {}) // Called after all dependencies have loaded
  .catch(function(error) {})
```

As you can see, the module loading mechanismn is asynchronous. However, synchronous access is also supported, provided the module has already been loaded:

``` JavaScript
var module = System.get('my-module').default
// Returns `undefined` if no module with that name has been loaded, yet
```

Also, there are of course import statements.

Using JavaScript's own module system and loader solves the namespacing issue and makes lazy loading the default as each module specifies its dependencies exactly and thus only the bits that are needed get loaded.

With this proposal it's then the responsibility of the Router to call `System.import().then()` to load all necessary modules for the current route. More on that further down in "Router DSL and active generation".

## New API

The new container doesn't have a resolver. Instead the module system is used for retrieving classes. There is no reason to hide a well working system. This also greatly simplifies the container API.

The new API I propose here uses named parameters for extra clarity. I somehow need to look up the parameter order everytime I use the current API.

### Object creation

Since the lookup of the class is done through the module system, there needs to be a way to apply dependency injections on object creation.

`containerTags` specified on the class itself determine which rules get applied. In this example, the class `SomeController` has the tag `'controller'` specified. Consequently when an instance of that class gets created using `container.createObject()`, it gets all injections for a controller (A tag based approach offers more flexiblity than the current type based approach while not being any more difficult to understand or manage).

``` JavaScript
// some-package/SomeController.js
var SomeController = Ember.Object.extend()
SomeController.reopenClass({ containerTags: ['controller'] })
export default SomeController
```

``` JavaScript
import SomeController from 'some-package/SomeController'

export default Ember.Object.extend({
  helloWorld() {
    var initalValues = { ... }
    var controller = this.container.createObject(SomeController, initalValues)
  }
})
```

### Singletons

Ember applications have objects called singletons that exist only once per application (Singletons can be realized using just the module system as well, but those are unique for the whole page and that's bad for testing). The container keeps track of all singletons.

``` JavaScript
import Store from 'ember-data'
...
var store = container.createObject(Store, optionalInitialValues)
container.registerSingleton('store', store)
container.getSingleton('store')
```

### Injections

The most common case is to inject a singleton as a property:

``` JavaScript
container.inject({
  into: 'route',
  as: 'store',
  fromSingelton: 'store'
})
```

Also classes can be injected, then for every new object a new instance gets created every time.

``` JavaScript
import Something from 'some-package/Something'
container.inject({
  into: 'route',
  as: 'store',
  from: Something
})
```

## Router DSL and active generation

Here's a possible solution on how `Controller`s and `Route`s modules can be associated to their respective URL in the `Router`.

Are there any suggestions about this?

``` JavaScript
Router.map(function() {
  this.route('menu') // Asumes `./menu.js` to exist
})
```

``` JavaScript
// menu.js
export Route = Ember.Route.extend()
// `export Controller` isn't there, so create one automatically
export default as template from './menu/template'
```

## Imports in templates

For lazy loading to work, templates need dependencies, too. They can't simply rely on that everything needed is already loaded. Ember's templates compile to JavaScript, so I propose to **use ES6 import statements for the dependencies in the template's compiled output**. Yes, that's a rather drastic change, but let's think about it. The main advantage is that a compiled template could be treated like any JavaScript module and it depends on any helpers, views, components or itemControllers used in it. The default export is the template function.

## Bundles

[SystemJS](https://github.com/systemjs/systemjs) is a project that builds on top of the es6 module loader and it adds support for bundles transparently:

``` JavaScript
System.bundles['my-bundle'] = ['jquery', 'bootstrap/js/bootstrap']
System.import('jquery') // Accesses `my-bundle.js`
```

JavaScript modules (and their dependencies) can easily be packaged into bundles using tools. (Like [systemjs-builder](https://github.com/systemjs/builder) used in [jspm](http://jspm.io/))

Note: Bundling JavaScript files won't be necessary in a few years from now. (With SPDY, HTTP/2)

# Drawbacks

- Introduces a new dependency: es6-module-loader polyfill
- Requires changes all over the place

# Alternatives

Another approach is to keep using es6 modules like we do today in ember-cli. However, this approach relies on loading every module up-front. This approach won't ever work for truly ambitious applications, though.

# Unresolved questions

- Dependencies in templates?
- How modules get referenced in `Router.map`
- Can backwards compatibility be ensured or is this an ember 2.0 thing?
