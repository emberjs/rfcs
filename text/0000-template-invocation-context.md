- Start Date: 2020-06-27
- Relevant Team(s): Ember.js, Learning
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Allow helpers, modifiers & components to access their template invocation stack in debug builds 

## Summary

Add a public API that allows a helper, modifier, or component to access information about the "template invocation stack" whereby it was invoked. This access would be for debug builds only and would be designed to allow helpers and modifiers to provide more useful error messages when invalid input is received or something goes wrong.

## Motivation

One of the most frustrating parts of the Ember developer experience today is that errors raised in helpers and modifiers often do not provide enough context for the developer to know where to address the issue.

For example, if the property you pass into a `foo` helper is expected to be a function but instead is `undefined` at runtime, you might have an error thrown `undefined is not a function`, with a javascript stack trace pointing to the `foo` helper implementation. Following the stack trace up to the template layer results in disappointment (even for a developers with a solid understanding of glimmer). If the author of `foo` is very conscientious, he or she might assert the presence and type of the expected function argument. This would result in a more readable error, but no improvement with respect to understanding in which template this particular instance of the helper was used. If the app has many usages of the `foo` helper, the developer not only has a problem, but has a problem resolving the the problem.

The `fn` and `on` helpers in Ember core were recently updated to address this issue. Prior to Ember 3.19, the framework would report, for example, that a function is expected as the first argument to `fn`, but not tell you anything that might help you identify *which* usage of `fn` was the problem. Thanks to the work of @rwjblue, [`fn` and `on` now provide helpful context about the template location where the problem is occurring](https://github.com/emberjs/ember.js/pull/18871). Unfortunately, the API used to do this is not public and is sufficiently complex to preclude most app or addon authors to provide error messages of similar usefulness for their own helpers and modifiers.

It is helpful to have the template path and location within that template of helper, modifier or component in question. However, a developer ideally wants to know the full “template invocation stack” of a given helper or modifier that has thrown an error. For example, consider the case of a helper that is only invoked from one component but that component is invoked from many places. Knowing the helper invocation location just isn’t enough, you need the full stack. i.e. the template location of the invocation, the parent template location of *that* location, and so forth, likely terminating with a route template. 

## Detailed design

There are two pieces to the API design.

1) What Javascript data structure should be used to represent the stack?
2) What Javascript API will developers use to access the stack info?

There is also an important implementation detail to consider:

3) How is the stack information is tracked by Glimmer and collected when requested via this API?

This section will take each of items in turn:

### 1) What Javascript data structure should be used to represent the stack?

This self-referential interface would be used:

```ts
interface TemplateLocationInformation {
  /**
    The module name of the template that invoked this component/helper/modifier.
  */
  template: string;

  /**
    The line of the invocation in the template. The line numbers in a template
    starts at `1`.
  */
  line: number;

  /**
    The column of the invocation in the template. The column numbers in a template
    start at `0`.
  */
  column: number;

  /**
    The location of the invocation of the "parent" of this component/helper/modifier.

    For example, if a route template invokes `foo-bar` component, the `parent` property
    inside `foo-bar` would reference the route template invocation information.
  */
  parent?: TemplateLocationInformation;


  /**
    A method returning a string representation suitable for console output.
  */
  toString(): string;
}
```

### 2) What Javascript API will developers use to access the stack info?

Determining a suitable API for accessing this information is a challenge. Helpers and modifiers can be written in a functional manner, which means there is no `this`, so a `getOwner(this)`-style API would seem to be insufficient. Indeed, the only surface area we have to work in functional helpers/modifiers are the `params` (array) and `named` (object) arguments. In class-based modifiers, the equivalent is `this.args.positional` and `this.args.named`. In class-based helpers, we have the the `params` (array) and `named` (object) arguments to the computed method. Finally, for components there are no positional arguments -- we only have named arguments available as `this.args`.

This suggests two possible approaches:

a) A new named argument could be included, a function `_getInvocationStack` that could be accessed as follows:

```js
// functional helper
// functional modifier
// class-based helper

let templateStack = named._getInvocationStack();

// class-based modifier

let templateStack = this.args.named._getInvocationStack();

// component

let templateStack = this.args._getInvocationStack();
```

In production, this method could return an empty object.

a') Same as (a) but with a property getter instead of a function reference: e.g. `let templateStack = named._invocationStack;`

b) A new debug function that can extract the stack from a hidden (symbol?) property stashed on the named/args object. It would look like this:

```js
import { getTemplateLocationInformation } from '@glimmer/debug';

// functional helper
// functional modifier
// class-based helper

let templateStack = getTemplateLocationInformation(named);

// class-based modifier

let templateStack = getTemplateLocationInformation(this.args.named);

// component

let templateStack = getTemplateLocationInformation(this.args);
```

In production, this method could return an empty object.

In either case, we could consider a lint rule to make sure this is invoked within an `if (DEBUG) { ... }` block.

### 3) How is the stack information is tracked by Glimmer and collected when requested via this API?

I need help on this section. From discussing briefly with @rwjblue, I have learned that one of the precursors for implementation of something like this is to ensure that Ember's internal template compilation is aware of debug vs prod builds (today we compile templates with the same set of built-in AST plugins). This would be beneficial to reducing production build size, because the lack of differentiation in the current system has caused template size bloat because the compiler must always include debug information.

## How we teach this

This should be included in the API docs for helpers, modifiers and components with example code shown, along with example output. If a configuration option is included like the one posited under "Alternatives," this should be included in the guides under "Configuration > Debugging".

When released, it should be included in the Ember.js Blog post for the release as well as highlighted in Ember Times. This could save existing Ember developers a lot of time and frustration.

## Drawbacks

APIs for helpers and modifiers are currently wonderfully simple and elegant. The addition this ugly debugging API compromises that design.

The fact that that this will only be available in debug builds adds to complexity for consumers of the API and could potentially lead to errors in production if a developer does not realize this.

## Alternatives

@rwjblue created an addon to address this problem: [ember-template-invocation-location](https://github.com/rwjblue/ember-template-invocation-location). This work is the basis for much of the content of this RFC. One alternative is to point the community toward this addon and not make any changes to core. However, @rwjblue, feels that this addon isn’t quite good enough. It requires each “layer” to have a dependency on it for it to function through the addon template layer, and that basically means that you are highly unlikely to consistently ensure your stack leads all the way back to the root.

In addition to (or perhaps instead of) providing this new invocation stack access to helpers, modifiers, and components in hopes that they will produce useful error messages, the framework could catch errors, decorate them with the template invocation stack and rethrow the error. This could potentially be controlled by a configuration setting similar to how backburner currently does it](https://guides.emberjs.com/release/configuring-ember/debugging/#toc_errors-within-emberrunlater-backburner).

Another possible alternative would be to provide a public API to emit the same hierarchical message that is used for the backtracking assertion (which is what the on and fn PR linked to above uses). This doesn’t solve the problem 100% (it wouldn’t directly provide access to the line and column info), but it’s fairly close and the plumbing is already largely available (still requires owner access though that might be mitigated by an argument as suggested in the api section). @rwjblue suggests this would likely be much easier to implement and could be improved over time to include line and column info when we have solved question 3 above (and in fact, would mean that all of Ember's other template error messages like the backtracking rerender one, or on/fn mentioned above, would get that info also).

Taking no action is not a good option as this is one of the most frustrating aspects of the modern Octane Ember developer experience.

We could consider whether something like [React's Error Boundaries](https://reactjs.org/docs/error-boundaries.html) would replace or augment the proposed approach.

## Unresolved questions

* (2) under Detailed design proposes two APIs. We need to choose one (or come up with something better).
* (3) needs to be expanded by someone who knows this portion of the Ember's implementation
* The Alternatives need weighing and the addition of rationale for not taking some of the listed paths.