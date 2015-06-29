- Start Date: 2015-06-28
- RFC PR:
- Ember Issue:

# Summary

Using an initializer customEvents can be added to an application.
An Ember CLI Addon may add customEvents to a consuming application.

# Motivation

At one point an initializer could be used to add custom events to the
application#customEvents object. This was not documented, but worked.

Advantages for adding customEvents using an initializer:

* An addon may defines customEvents to be used by an app, e.g. new events such
as gestures or other mobile events, perhaps a `fastClick` event added by an addon.

* Communication between DOM elements, Ember Components. Actions may not work
in some cases. There are use cases where it's nearly impossible to wire the
targetObject to a component that is nested below a chuck of DOM, perhaps DOM
elements that were added via rendering an `outlet` (new context).

# Detailed design

Instead of eagerly copying the customEvents property over to the instance before
initializers are run, set customEvents on the application after initializers are run.

No changes needed to the existing method for adding custom events:

```javascript
var App = Ember.Application.create({
  customEvents: {
    // add support for the paste event
    paste: 'paste'
  }
});
```

Example initializer augmenting the application's customEvents property:

```javascript
import Ember from 'ember';

export function initialize(registry, application) {
  var customEvents = application.get('customEvents') || {};

  Ember.String.w('toggle expand collapse').forEach(function (prefix) {
    var name = Ember.String.fmt("%@OffCanvas", prefix);
    customEvents[name] = name;
  });

  application.set('customEvents', customEvents);
}

export default {
  name: 'ember-off-canvas-components/custom-events',
  initialize: initialize
};
```

**Custom events** should be **first class citizens** in an Ember app and in
Ember Components that arrive to an app using an initializer (via an addon).
Custom events work great and can provide a solid solution for communicating
between DOM elements (components) in a scenario where the components have a
common parent element and otherwise communication would be impossible.

# Drawbacks

Addon developers should name the events in a unique manner, to minimize
the possibility of conflicting with events added by other addons.

# Alternatives

By not making customEvents initializable, addon developers cannot easliy use a
standard feature supported by browsers, custom events.

* When using custom events in an addon, the consuming application is required
to setup the custom events on the application (additional ceremony, not a
straight forward installation of the addon).

# Unresolved questions

There must have been a reason this behavior changed, perhaps due to FastBoot.

* See [FastBoot changes seem to break setting Application#customEvents within initializer][10534]

[10534]: https://github.com/emberjs/ember.js/issues/10534


