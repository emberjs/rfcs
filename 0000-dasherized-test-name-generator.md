- Start Date: 2017-03-20
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Change test name humanization (see [here](https://github.com/ember-cli/ember-cli-test-info/blob/master/index.js#L27)).

Examples for `some-multi-word-service-test.js` :

* Old name: `Unit | Service | some multi word service`
* New name: `Unit | Service | some-multi-word-service`

# Motivation

The use of humanized name tests make filtering more confusing for newcomers. If someone starts testing an already large application, chances are he/she will want to filter his/her tests to run only the ones they're working on. By default, jshint tests are run along the tests themselves.

Say a user is running tests for `some-component`. When he/she runs `ember test --filter some-component`, **they will see the jshint tests run** (and usually, pass), but they **won't see** their own tests run. This creates a confusing false positive situation where you're believing your tests are passing when they clearly shouldn't be.

Moreover, the use of a third way of referring to a component/module in addition to `foo-bar` and `FooBarComponent` is cumbersome. *It is to be determined whether or not the community actually relies on the humanized form for test names (see [Unresolved questions](#unresolved-questions)).*
As a very minor point, this humanized form is also slightly less handy to use with `--filter`, as you're required to use quotes, whereas this is not a problem with a dasherized name (`--filter "some name"` vs `--filter some-name`).

# Detailed design

*/!\ This probably needs to be completed by someone other than me :)*

The humanize logic is in [ember-cli-test-info](https://github.com/ember-cli/ember-cli-test-info/blob/master/index.js#L41). In `index.js`, we may change the `_friendlyText` function:

```javascript
// Current _friendlyText function
_friendlyText: function(text, testType, blueprintType) {
    var ret = testType + SEPARATOR;
    if (blueprintType) {
      ret += blueprintType + SEPARATOR;
    }
    ret += this.humanize(text);
    return ret;
}
  
// Proposed _friendlyText function
_friendlyText: function(text, testType, blueprintType) {
    var ret = testType + SEPARATOR;
    if (blueprintType) {
      ret += blueprintType + SEPARATOR;
    }
    ret += text;
    return ret;
}
```

I think we may also remove the `humanize` function if it isn't used elsewhere.

# How We Teach This

For newcomers, there is rather less to be taught as the dasherized name is the default. As stated earlier, this change would mean one less naming form, which makes testing slightly more approachable.

However, as the default behavior changes, it should be made clear in the release notes, so that current users don't get confused when generating tests.

# Drawbacks

The current behavior of test naming is changed, which potentially means confusion for current users who filter their tests with spaces, as a newly generated test won't be ran with a filter which is valid for previous tests. However, the output will then be less confusing than the current situation (see [Motivation](#motivation)), as with such a filter **no test will run**, making the naming difference clearer.

# Alternatives

The main alternative is leaving the name humanized.

# Unresolved questions

* How commonly is the humanized form of test names relied upon in Ember's community? How large would the impact of the change be if it was adopted in a future version of Ember? How can we measure it?
