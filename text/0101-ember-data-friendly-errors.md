---
Start Date: 2015-10-23
RFC PR: https://github.com/emberjs/rfcs/pull/101
Ember Issue: https://github.com/emberjs/data/pull/3930

---

# Summary

Add more illustrative detail to the default Ember Data Adapter Errors.

# Motivation

With a production Ember project, it's common to have many errors of the form "Adapter Error",
originating from deep in the Ember Data stack and carrying little context about what the
original error cause was.

The intent is to add the original request URL, the response code, and some payload information
to the default Error message for `DS.AdapterError`s. From there Errors can be handled or
tracked as normal.

# Detailed design

I've been using something similar to the following Adapter (`friendly-error-adapter.js`):

```js
import ActiveModelAdapter from 'active-model-adapter';

import DS from 'ember-data';

export default ActiveModelAdapter.extend({

  ajax(url, method)  {
    this.lastRequest = {
      url:    url,
      method: method
    };
    return this._super(...arguments);
  },

  handleResponse: function (status, headers, payload) {
    let payloadContentType = headers["Content-Type"].split(";").get("firstObject");
    let shortenedPayload;

    if (payloadContentType === "text/html" && payload.length > 250) {
      shortenedPayload = "[omitted long blob of HTML]";
    } else {
      shortenedPayload = payload;
    }

    let errorMessage = `Ember Data Error (${this.lastRequest.method} ${this.lastRequest.url} returned a ${status}). \n Payload (${payloadContentType}): \n\n ${shortenedPayload}`;

    if (this.isSuccess(status, headers, payload)) {
      return payload;
    } else if (this.isInvalid(status, headers, payload)) {
      return new DS.InvalidError(payload.errors, errorMessage);
    }

    let errors = this.normalizeErrorResponse(status, headers, payload);

    return new DS.AdapterError(errors, errorMessage);
  }
});
```

(Note that the code inside the adapter could be MUCH simpler and cleaner, the above
is a very quick hacked up example! :bomb:)

The intent is to get an error message out of the form:

 1. "Ember Data Error"
 2. Request Method & URI
 3. Response Status
 4. Response Content Type
 5. A sane representation of the Response payload

# Drawbacks

Adding complexity to an Error handler always runs the risk of generating errors inside
the handler itself, which would not be overly friendly.

# Alternatives

There's probably quite a few different pieces of information that could be included
in the message.

We could also potentially look at attaching the extra information to other fields on
the `AdapterError` (and its subclasses). The only drawback there would be that most
error reporters would then not include that information by default.

# Unresolved questions

 * Exact Error Message Format
