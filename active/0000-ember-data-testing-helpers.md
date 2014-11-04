- Start Date: (fill me in with today's date, 2014-11-13)
- RFC PR: (leave this empty)
- Ember Data Issue: (leave this empty)

# Summary

Remove the `DS.FixtureAdapter` and replace with test helpers.

# Motivation

We have not done a great job at making the FixtureAdapter a pleasant
alternative to stubbing AJAX (or other transport mechanisms such as
`localStorage/indexedDB`). There are some leaks in the current FixtureAdapter
implementation:

* Ember Data's internals tend to mutate existing payloads. This is especially
the case for passthrough attributes. For example, you may have `taco:
DS.attr()` and pass `{flavor: 'dorito'}` as the value for that attribute.
Throughout testing your app, or sometimes even loading the record itself,
this data can get mutated. Ember Data's tendency to mutate payloads causes
strange race conditions in your test that can often be hard to debug.
* The FixtureAdapter does a poor job at managing state. Like the above point,
it will mutate the `FIXTURES` array directly. This makes it inconvenient as
loading things into the store, or changing/deleting records, will mutate the
`FIXTURES` array directly, giving you no point of reference to the original
values unless the caller explicitly calls some sort of clone operation (such
as `Ember.copy(object, true)` or `$.extend(target, source, true)`).
* Your fixtures don't represent your app's actual payload. Sometimes the
FixtureAdapter is bad at wiring up the correct adapter or serializer (in my
experience, this may have changed lately). The disconnect between reality and
test can lead to some confusion at 3AM when things don't work in production.

While there are available options for full-stack style integration tests, such
as [ic-ajax][ic-ajax], [Pretender][pretender], and [SinonJS][sinon], the
options for Ember Data at the unit level currently aren't great. It is also
currently impossible to access the store without reaching into
`App.__container__`. For example, consider the following code:

```javascript
describe("settings page", function(){
  describe("when the user has payment access", function(){
	  beforeEach(function(){
	    this.server.get('/api_settings', function(){
	      var statusCode = 200;
	      var headers = {"Content-Type": "application/json"};
	      var settingsResponse = {user: {has_access: true}};
	      return [statusCode, headers, JSON.stringify(settingsResponse)];
	    });
	    return visit('/settings');
	  });
	  it("shows the credit cards", function(){
      assert.equal(find(".spec-credit-cards").length, 4);
	  });
	});


  describe("when the user does not have payment access", function(){
    describe("when the user has payment access", function(){
      beforeEach(function(){
        this.server.get('/api_settings', function(){
          var statusCode = 200;
          var headers = {"Content-Type": "application/json"};
          var settingsResponse = {user: {has_access: true}};
          return [statusCode, headers, JSON.stringify(settingsResponse)];
        });
        return visit('/settings');
      });
      it("does not show the credit cards", function(){
        assert.equal(find(".spec-credit-cards").length, 0);
      });
    });
  });
});
```

You'll notice that there's a lot of *noisy* code. By noisy, I mean code that
doesn't matter but is required for the test to work (such as the url, the
content headers, status code, etc). If we could get access to the user record
directly, we could reduce a lot of noisy code, making the tests more readable
and easier to maintain over time:

```javascript
describe("settings page", function(){
  describe("when the user has payment access", function(){
    beforeEach(function(){
      getRecordById('user', 1).set('hasAccess', true);
      return visit('/settings');
    });
    it("shows the credit cards", function(){
        assert.equal(find(".spec-credit-cards").length, 4);
    });
  });
  describe("when the user has payment access", function(){
    beforeEach(function(){
      getRecordById('user', 1).set('hasAccess', false);
      return visit('/settings');
    });
    it("shows the credit cards", function(){
      assert.equal(find(".spec-credit-cards").length, 4);
    });
  });
});
```

# Detailed design

## Move FixtureAdapter to its own repository and bower repo

This allows people relying on the FixtureAdapter to continue using it. Perhaps
the FixtureAdapter itself will find a new steward such as @kurko has taken over
the LocalStorage, IndexedDB, and JSONAPI adapters.

## Make a public test helper for public methods on the store

Ideally, Ember Data users should not need direct access to the store to be
productive in their testing. We can provide some helpers that use the word
"record" in the helper name to indicate it came from Ember Data. Some method
name proposals for the current [public store API][ember-data-public-api]:

* `allRecords(type)` - `store.all(type)`
* `createRecord(type, attributes)` - `store.createRecord(type, attributes)`
* `deleteRecord(record)` - `store.deleteRecord(record)`
* `findRecord(type, id, preload)` - `store.find(type, id, preload)`
* `getRecordById(type, id)` - `store.getById(type, id)`
* `hasRecordForId)type, id)`- `store.hasRecordForId(type, id)`
* `metaForRecordType(type, metadata)` - `store.metaForType(type, metadata)`
* `metadataForRecordType(type)` - `store.metadataFor(type)`
* `normalizePayload(type, payload)` - `store.normalize(type, payload)`
* `pushRecord(type, data)` - `store.push(type, data)`
* `pushManyRecords(type, recordDatas)` - `store.pushMany(type, recordDates)`
* `recordIsLoaded(type, id)` - `store.recordIsLoaded(type, id)`
* `unloadAllRecords(type)` - `store.unloadAll(type)`
* `unloadRecord(type)` - `store.unloadRecord(type)`

## Implementation

We could use Ember's `registerTestHelper` and `registerAsyncHelper` methods.
http://emberjs.com/api/classes/Ember.Test.html#method_registerHelper

# Drawbacks

May expose more of the store API than we want for tests.

# Alternatives

* Fixing the FixtureAdapter. We brought this up in an Ember Data meeting and
decided against it.
* Use ic-ajax/Pretender/SinonJS style integration mocks for all ember data
testing. These fixtures quickly become difficult to maintain and they are
quite verbose.

# Unresolved questions

* Unit Testing Serializers / Adapters - Is this something people want?

<!-- Links -->
[ic-ajax]: https://github.com/instructure/ic-ajax
[pretender]: https://github.com/trek/pretender
[sinon]: http://sinonjs.org/
[ember-data-public-api]: http://emberjs.com/api/data/classes/DS.Store.html
