- Start Date: 2015-06-29
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Actions are a way to extract business logic and isolate them. The corresponding pattern in Rails is [Service Objects](https://blog.engineyard.com/2014/keeping-your-rails-controllers-dry-with-services).

# Motivation

Extracting business logic into functions has the following benefits to larger applications:

- Business logic, which usually handles complex cases, is isolated in tests.
- Actions can be stubbed out with simple responses that reduce the friction of testing route / component actions.
- Since actions are functions, they can be composed to create more complex actions.
- Isolating business logic from the route / component layer means that you can focus on how the application should respond to the results of the action rather than the business logic itself.

# Detailed design

An action must be defined by exporting a function under the `actions` directory:

`actions/update-credit-card.js`
```javascript
import Ember from 'ember';

const { RSVP } = Ember;

export default function (creditCard) {
  let { resolve, reject, promise } = RSVP.defer();
  $.get('/api/chargify_direct').then(resolve, reject);
  return promise.then(function (json) {
    let { resolve, reject, promise } = Ember.defer();
    $.post(json.url, creditCard).then(resolve, reject);
    return promise;
  });
}
```

If the action requires injections to perform the action, it must export an object that responds to `execute`:

`actions/upload-image.js`
```javascript
export function execute (file) {
  return file.upload().then((response) => {
    let image = this.store.createRecord('image', {
      name: get(file, 'name'),
      size: get(file, 'size'),
      url: response.headers.Location
    });
    return image.save();
  });
}

export default {
  store: Ember.inject.service(),
  execute: execute
}
```

Actions should **always** run in an isolated context. This context is either an empty object in the case of a function being exported, or the host object if an object is exported.

Action lookup should be lazy, but the injection itself should not be. This is so developers that use actions should be able to call them like they are functions available on the host object, but are able to test them in isolation by stubbing them out (as prescribed in the testing section):
 `routes/account.js`
```javascript
import Ember from 'ember';

export default Ember.Route.extend({
  updateCreditCard: Ember.inject.action(),
  actions: {
    updateCreditCard(form) {
      return this.updateCreditCard(form).then(() => {
        this.modelFor(this.routeName).refresh();
      });
    }
  }
});
```

Injected actions should desugar to something like:

```javascript
{
  updateCreditCard: function (...args) {
    let action = this.container.lookup('actions:update-credit-card');
    if (action.execute) {
      return action.execute(...args);
    } else {
			return action(...args);
    }
  }
}
```

### Testing

Stubbing out actions should be available under testing mode, through a method called `stubAction`.

Component integration tests should have a method called `stubAction` on the context object during setup and tests:

`tests/unit/components/type-ahead-test.js`
```javascript
import Ember from 'ember';
import { moduleForComponent, test } from 'ember-quint';

moduleForComponent('type-ahead', {
  integration: true,
  beforeEach() {
    this.render('{{type-ahead url="/fetch-completions"}}');
  }
});

test('fetching data from a remote data source', function (assert) {
  this.stubAction('fetchTypeAheadCompletions').with(function (url) {
    assert.equal(url, '/fetch-completions');
    return Ember.RSVP.resolve([
      'New York',
      'New Jersey'
    ]);
  });
});
```

For acceptance tests, the `stubAction` method should be available on the application instance:

```javascript
import Ember from 'ember';
import { module, test } from 'unit';
import startApp from '../helpers/start-app';

var App;
module('account', {
  beforeEach() {
    application = startApp();
  },
  afterEach() {
    Ember.run(application, 'destroy');
  }
});

test('updating credit card information', function (assert) {   visit('/account');
  application.stubAction('updateCreditCard', function (creditCard) {
    assert.deepEqual(creditCard, {
      cardNumber: '424242424242',
      cardholderName: 'Tomster',
      cvv: '930',
      expiryMonth: 3,
      expiryYear: 2016
    });
    return Ember.RSVP.resolve();
  });

  fillIn('#card-number', '424242424242');
  fillIn('#cardholder-name', 'Tomster');
  fillIn('#cvv', '930');
  fillIn('#expiry-month', '03');
  fillIn('#expiry-year', '2016');
  click('#save');
});
```

# Drawbacks

Added complexity to apps. This is not intended for smaller sized applications. Rather, this is for applications on the larger side that have lots of moving parts and interactions.

# Alternatives

The alternative is to use services as-is and override the `create` method to return a method. This doesn't provide the same level of support, but does provide a path forward.

Without actions, it is hard to write tests for objects that involve complicated business logic without a great deal of setup and network simulation.

# Unresolved questions

Discussion of other possible use cases.