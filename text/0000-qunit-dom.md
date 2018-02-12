- Start Date: 2018-02-12
- RFC PR: (leave this empty)

# Summary

Introduce [`qunit-dom`] as a dependency by default in the `app` and `addon` blueprints.

[`qunit-dom`]: https://github.com/simplabs/qunit-dom

# Motivation

> Why are we doing this?

QUnit assertions are quite verbose and can look like this:

```js
assert.equal(this.element.querySelector('.title').textContent.trim(), 'Hello World!');
```

with `qunit-dom` we can write much more readable assertion for DOM elements:

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


# Alternatives

> What other designs have been considered?

None

> What is the impact of not doing this?

We will keep using ugly, hard-to-read assertions by default and leave it up
to our users to discover `qunit-dom` by themselves.


# Unresolved questions

None
