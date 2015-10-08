- Start Date: 2015-10-08
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Service methods provide a way to isolate business logic from your app and test it in isolation from the rest of your app.

# Motivation

I've found that when writing larger apps, user stories appear that are complicated to implement, but are simple in description. An example of a user story I've encountered is:

> As a user, I would like to add multiple artworks to an invoice.

This initially was developed as an action in a route:

```javascript
import Ember from 'ember';

export default Ember.Route.extend({
  actions: {
    addArtworksToInvoice(selection) {
      let store = this.store;
      let invoice = store.createRecord('invoice');

      invoice.save().then(function () {
        return selection.fetch();
      }).then(function (artworks) {
        return artworks.map(function (artwork, idx) {
          return store.createRecord('invoiceItem', {
            artwork,
            invoice,
            position: idx
          });
        });
      }).then(function (items) {
        return Ember.RSVP.all(items.invoke('save'));
      }).then(() => {
        this.transitionTo('invoice', invoice);
      });
    }
  }
});
```

This ends up being a lot of code that interacts with the network, which for me, is painful (and slow) to test. Extracting the business logic of the story (the store mechanisms) into a stateless method makes the code:

- easier to test (by being able to mock out the service)
- have clearer intention

# Detailed design

The proposed pattern to ameliorate this is a syntatic sugar on top of Ember's existing service infrastructure. It adds two methods:

- `Ember.Service.method`
- `Ember.inject.method`

`Ember.Service.method` is a shorthand much like `Ember.Helper.helper` that takes a function. The method may be called by using `Ember.inject.method` on a host object. The previous example can be decomposed into these primitives now:

```javascript
import Ember from 'ember';

export default Ember.Service.method(function (store, selection) {
  let invoice = store.createRecord('invoice');

  invoice.save().then(function () {
    return selection.fetch();
  }).then(function (artworks) {
    return artworks.map(function (artwork, idx) {
      return store.createRecord('invoiceItem', {
        artwork,
        invoice,
        position: idx
      });
    });
  }).then(function (items) {
    return Ember.RSVP.all(items.invoke('save'));
  }).then(function () {
    return invoice;
  });
});
```

```javascript
import Ember from 'ember';

export default Ember.Route.extend({

  addArtworksToInvoice: Ember.inject.method(),

  actions: {
    addArtworksToInvoice(selection) {
      this.addArtworksToInvoice(this.store, selection).then((invoice) => {
        this.transitionTo('invoice', invoice);
      });
    }
  }
});
```

This example could be simplified further. Since service methods are a layer of sugar on top of services, these methods can obtain access to other services. This creates the ability to chain service methods and to remove boilerplate code (in the previous example, access to the store).

A service that is friendly with `Ember.inject.method` looks like:

```javascript
import Ember from 'ember';

export default Ember.Service.extend({
  store: Ember.inject.service(),

  execute(selection) {
    let store = Ember.get(this, 'store');
	  let invoice = store.createRecord('invoice');

    invoice.save().then(function () {
      return selection.fetch();
    }).then(function (artworks) {
      return artworks.map(function (artwork, idx) {
        return store.createRecord('invoiceItem', {
          artwork,
          invoice,
          position: idx
        });
      });
    }).then(function (items) {
      return Ember.RSVP.all(items.invoke('save'));
    }).then(function () {
      return invoice;
    });
  }
});
```


The interaction can now be tested without querying the server:

```javascript
module('/artworks', {
  integration: true
});

test('adding multiple artworks to an invoice', function (assert) {
  this.register('service:add-artworks-to-invoice', Ember.Service.method(function () {
    return Ember.RSVP.resolve();
  })
  visit('/artworks');
  click('#artwork-1');
  click('#artwork-4');
  click('#artwork-5');
  click('#artwork-6');

  click('#add-artworks-to-invoice');
  andThen(() => {
    assert.equal(this.currentPath(), 'invoice';)
  });
});
```

The ability to stub out the business logic in tests makes for testing the multitude of cases clearer and localizes what is being tested.

For a complete implementation of this pattern, see https://github.com/tim-evans/ember-service-methods.

# Drawbacks

This benefits a small subset of users, and is already available as an addon. It would be nice to have this story fleshed out for the rest of the community, especially those dealing with larger apps

# Alternatives

Another solution was considered, but did not reach consensus: https://github.com/emberjs/rfcs/pull/71. Other alternatives to this pattern are welcome.

# Unresolved questions

N/A
