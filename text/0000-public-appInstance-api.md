- Start Date: 2016-08-19
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Expose a public API that will give access to an Ember ApplicationInstance. This would allow external functions to lookup and invoke actions defined in an Ember Application.

# Motivation

Embedding Ember applications in existing apps works wonderfully and is a huge asset to those of us who wish to incrementally replace an existing UI with standalone Ember Apps with the eventual goal of using Ember Engines as a full replacement of the existing app.  However, simply embedding Ember Applications does not address the need of integrating these apps with the existing application.

To describe the motivation behind this request, please picture a large enterprise application written in some other framework.  In my case, these apps are written in GWT (Google Web Toolkit). Some of the existing features are easily replaced with standalone embedded Ember Applications.  There are other parts of the application that are not so simple to replace as the existing application must be able to invoke actions on an embedded Ember app.

For example, I have an existing button implemented in GWT that invokes some actions (also implemented in GWT), which in turn invoke additional actions via callbacks.  The feature I am replacing is one of these actions and will also fire a callback when the Ember action is complete.

Currently, the only way I have found to accomplish this is by going through some obviously deprecated and private APIs.  Exposing a public API to access an ApplicationInstance would be a fantastic way (I think) to allow more developers to *integrate* Ember applications with existing UIs vs simply *embedding* them.


# Detailed design

Example use of private and deprecated APIs to invoke an action on an embedded Ember application:
  ```
var myapp = require("myapp/app")["default"].create({
  "name": 'myapp',
  "autoboot": true,
  "rootElement": "#myapp"
});

myapp.__deprecatedInstance__.__container__.lookup('route:index').send('myAction');
  ```

As I am not familiar with why there is a `__deprecatedInstance__` or what `globalsMode` is or how that effects the decisions made around this request, I would simply invite the core developers to consider a public API for integration of Ember Applications, whatever that might look like.

# How We Teach This

Should the new feature be implemented like the currently private APIs, I would think this would be a matter of some simple examples on how to call into these APIs from the outside of an Ember Application.

# Drawbacks

The one drawback I can think of using the example provided in the Design section is that the Ember app may not currently be on the route for which the action is being invoked.  In my case, this is a single (index only) route application so I'm confident that I'm invoking my action in the right context.

# Alternatives

I have explored the `visit` API a little but was unable to get the same results I did when using the above private APIs.  This feels most useful in the context of testing but I could see how this API might be used as integration as long as actions could be invoked from the outside on some visited route/controller.

# Unresolved questions

- Are there any other strategies for integrating Ember Applications I am not aware of?  
- Why are `ApplicationInstance`s and `container`s private?

Thanks!
