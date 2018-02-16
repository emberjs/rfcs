- Start Date: 2018-02-12
- RFC PR: (leave this empty)

# Summary

Introduce [`qunit-dom`] as a dependency by default in the `app` and `addon` blueprints.

[`qunit-dom`]: https://github.com/simplabs/qunit-dom

# Motivation

> Why are we doing this?

In a modern Ember application making assertions around the state of the DOM is
fundamental to confirming your applications functionality. These assertions are
often quite verbose:

```js
assert.equal(this.element.querySelector('.title').textContent.trim(), 'Hello World!');
```

Using the `find()` helper of `@ember/test-helpers` we can simplify the DOM
element lookup, but the signal-to-noise ratio of the code is still not great:

```js
assert.equal(find('.title').textContent.trim(), 'Hello World!');
```

With `qunit-dom` we can write much more readable assertions for DOM elements:

```js
assert.dom('.title').hasText('Hello World!');
```

> What use cases does it support?

It supports the most common assertions on DOM elements, like:

- what text does the element have?
- what value does the `<input>` element have?
- is a certain CSS class applied to the element

The full API is documented at <https://github.com/simplabs/qunit-dom/blob/master/API.md>.

> What is the expected outcome?

Using `qunit-dom` will lead to more simple and readable test code.


# Detailed design

The necessary changes to `ember-cli` are relatively small since we only need
to add the dependency to the `app` blueprint, and the `addon` blueprint will
inherit it automatically.

This has the advantage (over including it as an implicit dependency), that
apps and addons that don't want to use it for some reason can opt-out by
removing the dependency from their `package.json` file.

A WIP pull request has been created already at <https://github.com/ember-cli/ember-cli/pull/7605>.


# How We Teach This

> Would the acceptance of this proposal mean the Ember guides must be
> re-organized or altered? Does it change how Ember is taught to new users
> at any level?

Once we decide that this is the right way to go, we should update the official
Ember.js testing guides to use `qunit-dom` assertions by default. This has the
nice side effect of making the testing code in the guides easier to read too.

At the same time (same minor release) we should update the relevant blueprints
in the `ember-source` package to use `qunit-dom` by default. This should be a
relatively small change as only the `component` and `helper` tests use
DOM assertions.

> How should this feature be introduced and taught to existing Ember
> users?

We should also explicitly mention this change in the release blog post and
recommend that people use this from now on. For those users that want to
migrate their existing tests to `qunit-dom` a basic codemod exists at
<https://github.com/simplabs/qunit-dom-codemod>.


# Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
> on the integration of this feature with other existing and planned features,
> on the impact of the API churn on existing apps, etc.
>
> There are tradeoffs to choosing any path, please attempt to identify them here.

- `qunit-dom` is "owned" by a third-party consulting company (simplabs) and
  the Ember CLI team is not directly in control.

- `qunit-dom` has not reached v1.0.0 yet so there might be small breaking
  changes in the future.

- `qunit-dom` is another abstraction layer on top of the raw QUnit assertions
  which adds to the existing learning curve.
  
- Adding `qunit-dom` to the default blueprint could make it look even more like
  `ember-mocha` is only a second-class citizen. Since we add it to the default
  `package.json` file it is easy to opt-out though and can be replaced with
  `chai-jquery` or `chai-dom` for a roughly similar API.


# Alternatives

> What other designs have been considered?

- Using the `find()` helper functions can be considered an alternative, but
  as mentioned above they still result in more verbose code than using
  `qunit-dom`. Another advantage is that `qunit-dom` generates a useful
  assertion description by default, while `assert.equal()` will just show
  something like "A does not match B". 

> What is the impact of not doing this?

We will keep using hard-to-read assertions by default and leave it up to our
users to discover `qunit-dom` by themselves.


# Unresolved questions

- Should the `ember-source` blueprints detect `qunit-dom` usage and fallback
  to raw QUnit assertions if the dependency can't be found?
