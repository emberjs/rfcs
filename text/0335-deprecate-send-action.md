---
Start Date: 2018-05-29
RFC PR: https://github.com/emberjs/rfcs/pull/335
Ember Issue: (leave this empty)

---

# Deprecate `.sendAction`

## Summary

In old versions of Ember (< 1.13) `component#sendAction` was the only way for a component to call an
action on a parent scope. In 1.13 with the so called _closure actions_ a more intuitive and flexible
way of calling actions was introduced, yielding the old way redundant.

## Motivation

With the new _closure actions_ being the recommended way, `component#sendAction` is not even
mentioned in the guides.
With the goal of simplifying the framework I think we should remove what is not considered the
current best practice.
_Closure actions_ have been available since 1.13. That is 3 years ago, so deprecating `sendAction`
should not cause too much pain and yet addons can support still support the last version of the 1.X
cycle if they really want to.

It is out of the scope of this RFC to enumerate the reasons why _closure actions_ are preferred over
_sendAction_ but you can find an in depth explanation of _closure actions_ in [this blog post from 2016](http://miguelcamba.com/blog/2016/01/24/ember-closure-actions-in-depth).

## Detailed design

A deprecation message will appear when `sendAction` is invoked. The feature will be removed in
Ember 4.0. The deprecation message will use the arguments passed to `sendAction` to generate a dynamic
explanation that will make super-easy for developers to migrate to closure actions.

As it is mandatory with new deprecations, a new entry in the deprecation guides will be added
explaining the migration path in depth.

To refresh what the migration path would look like in the typical use case.

**BEFORE**
```js
// parent-component.js
export default Component.extend({
  actions: {
    sayHi() {
      alert('Hello user!');
    }
  }
})
```

```hbs
{{!-- parent-component.hbs --}}
{{child-component salute="sayHi"}}
```

```js
// child-component.js
export default Component.extend({
  actions: {
    sendSalute() {
      this.sendAction('salute');
    }
  }
});
```

```hbs
{{!-- child-component.hbs --}}
<button {{action "sendSalute"}}>Send salute</button>
```

**AFTER**
```js
// parent-component.js
export default Component.extend({
  actions: {
    sayHi() {
      alert('Hello user!');
    }
  }
})
```

```hbs
{{!-- parent-component.hbs --}}
{{child-component salute=(action "sayHi")}}
```

```js
// child-component.js
export default Component.extend({
  actions: {
    sendSalute() {
      this.salute();
      // if the salute action is optional you'll have to guard in case it's undefined:
      // if (this.salute) {
      //   this.salute()
      // }
      // 
      // Alternatively, you can also define a noop salute function:
      // salute() {}
      //
      // This allows you to remove the guard while provinding an obvious place to add
      // docs for that action.
    }
  }
});
```

```hbs
{{!-- child-component.hbs --}}
<button {{action "sendSalute"}}>Send salute</button>
```

However _closure actions_ allow to be less verbose, so the same behavior could be attained using
less intermediate calls

```js
// parent-component.js
export default Component.extend({
  actions: {
    sayHi() {
      alert('Hello user!');
    }
  }
})
```

```hbs
{{!-- parent-component.hbs --}}
{{child-component salute=(action "sayHi")}}
```

```hbs
{{!-- child-component.hbs --}}
<button onclick={{@salute}}>Send salute</button>
```

## How we teach this

There are no new concepts to teach, but the removal of an old concept now considered outdated.

## Drawbacks

There might be some churn following the deprecation, specially comming from addons that haven't been
updated in a while.
Addons that want to support the latest versions of Ember without deprecation messages and still work
past Ember 1.13 will have to do some gymnastics to do so.

## Alternatives

Wait longer to deprecate it and keep `sendAction` undocumented until it's usage is yet more minoritary
than it is today, to lower the churn.

