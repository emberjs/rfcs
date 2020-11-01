---
Start Date: 2016-12-14
RFC PR: #191
Ember Issue: #14711

---

# Summary

We would like to deprecate and remove the **arguments** passed to the `didInitAttrs`, `didReceiveAttrs` and `didUpdateAttrs` component lifecycle hooks. These arguments are currently undocumented on purpose and considered a private API, imposes an unnecessary performance hit on *all* components whether they are used or not, and can be easily replicated by the users in cases where they are needed.

# Motivation

In the road leading up to Ember.js 2.0, [new lifecycle hooks](http://emberjs.com/blog/2015/06/12/ember-1-13-0-released.html#toc_component-lifecycle-hooks) were introduced to components in order to help users shift to a new mental model, dubbed Data Down Actions Up. The hooks were introduced by name, and their semantics explained, but there were no mentions of possible arguments received by them.

This lack of documentation for lifecycle hook arguments was deliberate. The hooks were introduced as an experiment with an eye to the then-upcoming angle bracket components, so the arguments to the hooks were considered private by the framework maintainers, as their design was still ongoing.

However, references to the lifecycle hook arguments started appearing in community resources. Users started betting on these arguments as the way forward, which in conjunction with the lack of an RFC process and clear messaging from the Ember.js maintainers lead to confusion.

This left the core team in a difficult position. Despite no longer endorsing lifecycle hook arguments, trying to communicate such could have the reverse effect by pointing a spotlight at them. The purpose of this RFC is then to clarify that lifecycle hook arguments have no future in the framework, and you should update your code to not make use of them.

The reason to officially deprecate lifecycle hook arguments is not only about messaging, but also because providing these arguments imposes an unnecessary performance penalty to every component in your application even if the arguments are not used.

To provide the arguments to the lifecycle hooks, Ember.js has to eagerly "reify" and save-off any passed-in attributes to allow diffing and construct several wrapper objects. In the few occasions where this logic is actually necessary, developers should be able to use programmatic patterns familiar to them and manually track changes as needed, as exemplified in the Transition Path section below.

# Transition Path

The transition path followed will be the standard one, which encompasses using the deprecation API to deprecate the feature and the related deprecation guide. While the lifecycle hooks share a deprecation identifier, they will be addressed in turn.

### `didInitAttrs`

Since this lifecycle hook is [already deprecated](http://emberjs.com/deprecations/v2.x/#toc_ember-component-didinitattrs), we suggest taking this chance to address two deprecations at the same time. Imagine you have a component that stores a timestamp when it's initialized for later comparison.

Before:

``` javascript
Ember.Component.extend({
  didInitAttrs({ attrs }) {
    this.set('initialTimestamp', attrs.timestamp);
  }
});
```
After:

``` javascript
Ember.Component.extend({
  init() {
    this._super(...arguments);

    this.set('initialTimestamp', this.get('timestamp'));
  }
});
```
### `didReceiveAttrs`

Let's say you want to animate a map widget from the old coordinates to the new coordinates.

Before:

``` javascript
Ember.Component.extend({
  didReceiveAttrs({ oldAttrs, newAttrs }) {
    if (oldAttrs && oldAttrs.coordinates !== newAttrs.coordinates) {
      this.map.move({ from: oldAttrs.coordinates, to: newAttrs.coordinates });
    }
  }
});
```
After:

``` javascript
Ember.Component.extend({
  didReceiveAttrs() {
    let oldCoordinates = this.get('_previousCoordinates');
    let newCoordinates = this.get('coordinates');

    if (oldCoordinates && oldCoordinates !== newCoordinates) {
      this.map.move({ from: oldCoordinates, to: newCoordinates });
    }

    this.set('_previousCoordinates', newCoordinates);
  }
});
```
### `didUpdateAttrs`

This hook is very similar to `didReceiveAttrs`, except it only runs on re-renders and not the initial render.

Before:

``` javascript
Ember.Component.extend({
  didUpdateAttrs({ oldAttrs, newAttrs }) {
    if (oldAttrs && oldAttrs.coordinates !== newAttrs.coordinates) {
      this.map.move({ from: oldAttrs.coordinates, to: newAttrs.coordinates });
    }
  }
});
```
After:

``` javascript
Ember.Component.extend({
  didUpdateAttrs() {
    let oldCoordinates = this.get('_previousCoordinates');
    let newCoordinates = this.get('coordinates');

    if (oldCoordinates && oldCoordinates !== newCoordinates) {
      this.map.move({ from: oldCoordinates, to: newCoordinates });
    }

    this.set('_previousCoordinates', newCoordinates);
  }
});
```
# How We Teach This

Due to the previous undocumented nature of the arguments, there is no official documentation that will require updating deprecated usage.

As required for framework deprecations, there will be a deprecation guide written up and linked from within the deprecation message. This deprecation guide will address the more common usage patterns associated with lifecycle hook arguments, such as the Transition Path example.

Additionally, the usage patterns present in the deprecation guide could also be documented in the component section of the official Guides, as a proactive approach for teaching newcomers.

# Drawbacks

One immediate drawback of this proposal is that due to references to the arguments in community resources, there are uses of them in the wild. Updating deprecated code will have to be done mostly manually, as automation might prove difficult.

Another drawback is that by the very nature of publishing this RFC, attention will be drawn to the arguments. It is our hope that the increased awareness will be a net positive due to the clear guidance gained by users of the framework.

It is then our assessment that these drawbacks are outweighed by the benefits of the change.

# Alternatives

There are two standout alternatives to the proposal presented here which are doing nothing, or making the arguments public and supporting them going forward, both of which are less than ideal for reasons stated previously.

Doing nothing would perpetuate the confusion surrounding lifecycle hook arguments. While it might be argued that that ship has sailed, we prefer to think that it's never too late to provide users of the framework with clearer messaging regarding usage of certain features.

Making the arguments public and supported would mean supporting APIs that did not go through the RFC process, meaning they do not align with some of the current values of the framework, nor would iteration on them would be possible without introducing breakage. Additionally, there are some performance penalties to supporting these arguments, mentioned in the Motivation section.

# Unresolved questions

None.
