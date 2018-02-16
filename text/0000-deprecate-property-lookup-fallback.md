- Start Date: 2-15-2018
- RFC PR:
- Ember Issue:

# Summary

Deprecate the fallback behavior of resolving `{{foo}}` by requiring the usage of `{{this.foo}}` as syntax to refer to properties of the templates's backing component. This would be the default behavior in Glimmer Components.

For example, given the following component class:

```js
import Component from '@ember/component';
export default Component.extends({
  init() {
    super(...arguments);
    this.set('greeting', 'Hello');
  }
});
```

One would refer to the `greeting` property as such:

```hbs
<h1>{{this.greeting}}, Chad</h1>
```

Ember will render "Hello, Chad".

# Motivation

Currently, the way to access properties on a components class is `{{greeting}}` from a template. This works because the component class is one of the objects we resolve against during the evaluation of the expression.

The first problem with this approach is that the `{{greeting}}` syntax is ambiguous, as it could be referring to a local variable (block param), a helper with no arguments, a closed over component, or a property on the component class.

## Exemplar
Consider the following example where the ambiguity can cause issues:

You have a component class that looks like the following component and template:

```js
import Component from '@ember/component';
import computed from '@ember/computed';

export default Component.extend({
  formatName: computed('firstName', 'lastName', function() {
    return `${this.firstName} ${this.lastName}`;
  });
});
```

```hbs
<h1>Hello {{formatName}}!</h1>
```

Given `{ firstName: 'Chad', lastName: 'Hietala' }` Ember will render the following:

```html
<h1>Hello Chad Hietala!</h1>
```

Now some time goes on and someone adds a `formatName` helper that looks like the following:

```js
export default function formatName([firstName, lastName]) {
  return `${this.firstName} ${this.lastName}`;
}
```

Due to the fact helpers take precedence over property lookups our `{{formatName}}` now resolves to a helper. When the helper runs it doesn't have any arguments so our template now renders the following:

```html
<h1>Hello !</h1>
```

This can be a refactoring hazord and can often lead to confusion for readers of the template. Upon encountering `{{greeting}}` in a component's template, the reader has to check all of these places: first you need to scan the surrounding lines for block params with that name; next you check in the helpers folder to see it there is a helper with that name (it could also be coming from an addon!); finally, you check the component's JavaScript class to look for a (computed) property.

Like [RFC#0276](https://github.com/emberjs/rfcs/blob/68812bf2d439c6bb77ad491e0159b371b68c5c35/text/0276-named-args.md) made argument usage explicit through the `@` prefix, the `this` prefix will resolve the ambiguity and greatly improve clarity, especially in big projects with a lot of files (and uses a lot of addons).

As an aside, the ambiguity that causes confusion for human readers is also a problem for the compiler. While it is not the main goal of this proposal, resolving this ambiguity also helps the rendering system. Currently, the "runtime" template compiler has to perform a helper lookup for every `{{greeting}}` in each template. It will be able to skip this resolution process and perform other optimizations (such as reusing the internal [reference](https://github.com/glimmerjs/glimmer-vm/blob/master/guides/04-references.md)
object and caches) with this addition.

Furthermore, by enforcing the `this` prefix tooling like the [Ember Language Server](https://github.com/emberwatch/ember-language-server) does not need to know about fallback resolution rules. This makes common features like ["Go To Definition"](https://code.visualstudio.com/docs/editor/editingevolved#_go-to-definition) much easier to implement since we have semantics that mean "property on class".

# Transition Path
The `{{this.foo}}` syntax has always been supported Handlebars and thus can be used in every version of Ember, however no guides or documentation promote its usage. While applications can start migrating today to this more explicit syntax, we can surface information about what properties on the class were accessed allowing developers to get a concise list of expressions that should be migrated. This can be surfaced because we are already disambiguate the syntax at runtime.

Using this information, we will also have the ability to create a codemod that will help migrate templates.

# How We Teach This

`{{this.foo}}` is the way to access the properties on the component class. This also aligns to property access in JavaScript.

Since the `{{this.foo}}` syntax has worked in Ember.Component (which is the only kind of components available today) since the 1.0 series, we are not really in a rush to migrate the community (and the guides, etc) to using the new syntax. In the meantime, this could be viewed as a tool to improve clarity in templates.

While we think writing {{this.foo}} would be a best practice for new code going forward, the community can migrate at its own pace one component at a time. However, once the fallback functionality is eventually removed this will result in a "Helper not found" error.

## Syntax Breakdown
The follow is a breakdown of the different forms and what they mean.

- `{{@foo}}` is an argument passed to the component
- `{{this.foo}}` is a property on the component class
- `{{#with this.foo as |foo|}} {{foo}} {{/with}}` the `{{foo}}` is a local
- `{{foo}}` is a helper

# Drawbacks
The largest downside of this proposal is that it makes templates more verbose, causing developers to type a bit more. This will also create a decent amount of deprecation noise, although we feel like tools like [ember-cli-deprecation-workflow](https://github.com/mixonic/ember-cli-deprecation-workflow) can help mitigate this.

# Alternatives
This pattern of having programming model constructs to distinguish between of the backing class and arguments passed to the component is not unique to Ember.

## What Other Frameworks Do
React has used `this.props` to talk about values passed to you and `this.state` to mean data owned by the backing component class since it was released. However, this approach of creating a specific object on the component class to mean "properties available to the template", would likely be even more an invasive change and goes against the mental model that the context for the template is the class.

Vue requires enumeration of `props` passed to a component, but the values in the template suffer from the ambiguity that we are trying to solve.

Angular relies heavily on the dependency injection e.g. `@Input` to enumerate the bindings that were passed to the component and relies heavily on TypeScript to hide or expose values to templating layer with `public` and `private` fields. Like Vue, Angular does not disambiguate.

## Introduce Yet Another Sigil
We could introduce another sigil to remove ambiguity. This would address the concern about verbosity, however it is now another thing we would have to teach.

## Do Nothing
I personally don't think this is an option, since the goal is to provide clarity for applications as they evolve over time and to provide a more consise mental model.

# Unresolved questions
Soon...

