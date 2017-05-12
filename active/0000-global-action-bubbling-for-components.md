- Start Date: (2015-03-30)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Currently components don't bubble their actions up to the controllers or routes above them. To deal with this users in Ember 1.x have been using views or passing actions into the components. This is very cumbersome and confusing as is well demonstrated by the following situation:

Let's say I have an outlet for my modals in the application template so it's global, I have a global action 'openModal' the whole point is to be able to pop a modal open from anywhere which works well in normal templates. Yet triggering the action from my component using
```
this.sendAction('openModal')
```
will not work. I have to do this:
```
{{my-component openModal="openModal"}}
```
then in the component I have to do something like
```
<button {{action 'popSomething'}}>
```
then in the component's js I have to say:
```
actions: {
  popSomething: function () {
    this.sendAction('openModal');
  }
}
```
And if it's another tier down, I have to keep doing that. This is really, really less than ideal and very confusing.

# Motivation

Make components easy to use and understand.

# Detailed design

I expressed this issue in a conversation with @knownasilya and he made this suggestion, which is also what others and I had come up with while discussing this issue:

Either be able to specify on an action that it's global and meant to bubble all the way up like this:
```
{{action 'openModal' global=true}}
```
or be able to specify that all of a component's actions should bubble to the top:
```
{{my-component global=true}}
```

Without this functionality, in my opinion components are too isolated and that makes them undesirable to use in a large app. Now that components are becoming so central in 2.0 more often than not, you do have the need to talk to the routes above the component, often all the way to the top.

# Drawbacks

I can't think of drawbacks.

# Alternatives

I can't think of alternatives.

# Unresolved questions

I don't know what this takes internally within Ember and how it sits with the plans for routable components and actions in 2.0 , but since event bubbling to the top is the native behavior in Javascript, I don't imagine it's very hard.

I at least have not heard a clear proposal to make component actions bubble up, so here it is.
