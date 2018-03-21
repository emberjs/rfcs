- Start Date: 2017-10-16
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Unit testing `serialize()` is fairly straightforward; however, unit testing
`normalize*` functions is not so obvious, despite the fact they’re often great
candidates for unit tests.

# Motivation

Unit testing `serialize()` is fairly straight forward and an example is
provided when the serializer test is generated. However, it is not straight
forward to unit test the `normalize*` functions in serializers.

Both the `serialize` and `normalize` processes are great candidates for unit
testing. Usually, they can be pure functions where an example payload is passed
in and assertions can be made about the resulting JSONAPI structure.

Troubleshooting small issues (even typos) with serializers is tricky in
acceptance tests compared to the fast direct feedback you can get with a unit
test.

# Detailed design

I'd like to propose adding some test helper functions. These are some that
I think would be useful.

1. `normalizePayload` - Given an example response, return the JSONAPI that
   would be pushed in to the store.

   Here is an example test:

   ```js
   import { module, test } from 'qunit';
   import { setupTest } from 'ember-qunit';
   import { normalizePayload } from 'ember-data/test-helpers';

   module('Unit | Serializer | user', function(hooks) {
     setupTest(hooks);

     const EXAMPLE_QUERY_RECORD_RESPONSE = {
       data: [{
         id: '10',
         type: 'user',
         attributes: {
           name: 'Tomster',
         },
       }],
     };

     test('normalizeQueryRecord - it can turn the array we get back from the server into the single record that ember-data expects', function(assert) {
       const store = this.owner.lookup('service:store');
       const normalizedHash = normalizePayload(store, 'user', EXAMPLE_QUERY_RECORD_RESPONSE, 'queryRecord');

       assert.deepEqual(normalizedHash, {
         data: {
           id: '10',
           type: 'user',
           attributes: {
             name: 'Tomster',
           },
         },
       });
     });
   });
   ```

2. `pushPayload` - Given an example response, return the ember-data models it
   creates.

   Like `store.push`, this will return a single model or an array depending on
   the scenario.

   Here is an example test:

   ```js
   import { module, test } from 'qunit';
   import { setupTest } from 'ember-qunit';
   import { pushPayload } from 'ember-data/test-helpers';

   module('Unit | Serializer | user', function(hooks) {
     setupTest(hooks);

     const EXAMPLE_QUERY_RECORD_RESPONSE = {
       data: [{
         id: '10',
         type: 'user',
         attributes: {
           name: 'Zoey',
         },
       }],
     };

     test('normalizeQueryRecord - it can turn the array we get back from the server into the single record that ember-data expects', function(assert) {
       const store = this.owner.lookup('service:store');
       const record = pushPayload(store, 'user', EXAMPLE_QUERY_RECORD_RESPONSE, 'queryRecord');

       assert.deepEqual(record.get('name'), 'Zoey');
     });
   });

   ```

# How We Teach This

* These test helpers could be documented along with the other test helpers.
* The blueprint for serializer tests could include an example of testing
  normalization along with the current example testing serialization.

# Drawbacks

* The only drawback I can think of is that it increases the size of the testing API.

# Alternatives

* Do nothing
* Publish an addon with these helpers
* Make it easier to actually trigger the normalization process, so that it can
  be tested as easily as the serialization process.

# Unresolved questions

* I'm not attached to any of the specific names. Would different names be clearer?
* In what package would these actually live? I was assuming
  `ember-test-helpers`, but maybe they could actually be a part of the `ember-data` addon?
* Another helper that might be helpful is something to assert that the
  normalization result is valid JSONAPI. Would it make sense to include that here?
* Should these be globally available test helpers (like the acceptance test
  helpers)? or something that is imported from `ember-test-helpers` (like `ember-test-helpers/wait`)?
