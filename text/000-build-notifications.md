- Start Date: 2018-04-27
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Include `ember-cli-build-notifications` by default

## Summary

`ember-cli-build-notifications` uses `node-notifier` to provide cross platform system notifications for `ember-cli` build events (specifically on build success and build error).

![](https://github.com/pdud/ember-cli-build-notifications/raw/master/example.png)

Adding ember-cli-build-notifications as a default addon will increase developer productivity and improve the feedback cycle of day-to-day development. It will also assist new ember devs by quickly notifying them of errors they do not yet know to look at for. The default settings should be verbose, as that would be most helpful to new Ember developers. 

## Motivation

Confusion as a result of the state of the build consumes a surprising amount of developer cycles. For new developers, this can be especially frustrating: 

> There’s no ide integration that shows you errors without compilation. The most frustrating thing about this is that I would write broken handlebars, and my server would refresh, but nothing would change. Eventually, I would see the error in the terminal where I was serving Ember and fix it. [My first 6 months using Ember: A retrospective](https://medium.com/@Realrobwebb/my-first-6-months-using-ember-a-retrospective-a5ecf3259b09)

There are several parallel paths that we can take to communicate the state of a build. Recent developments in the new error page are one of them.  Having `ember-cli-build-notifications` alert on **all** builds is another. Not all problems result in a build error, so the absence of a notification is also a useful tool in debugging.

Some of these scenarios include: 
* Addons, like `ember-cli-typescript` or tools like `watchman` breaking, as well as saves linked pacakges not starting a new build
  * Not seeing a success/error notification when you normally do will indicate something is off
* Page refreshed not happening after successful builds due to livereload disconnecting
  * When you see a success message but the page hasn't refreshed you know it's time to refresh the page on your own
* Making only style changes and not realizing the new css was already pushed to the browser
  * Seeing the success notificaiton makes it plain that your changes are now present, rather than wondering a build error has occured

Build notifications give automatic insight into these scenarios in an unobtrusive way. 

## Detailed design

1. Include `ember-cli-build-notifications` as part of the default kit
2. Generate it’s config file at `{app}/config/build-notifications.js`
3. Configure it to notify on success and notify, including sound, on failure
```js
module.exports = {
  buildError: {
    notify: true,
    notificationOptions: {
      sound: true
    }
  },
  buildSuccess: {
    notify: true,
    notificationOptions: {
      sound: false
    }
  }
};
```

## How we teach this

The best thing about this approach is that it teaches itself. The alerts will arrive when they’re supposed to. The generated config file will expose the ability to configure it’s functionality. It also doesn’t need any special editor config to be useful to the developer and supports the major OSes.

## Drawbacks

* Some may find the notifications noisy or unnecessary. If so, they can easily tweak the config or remove the add-on
* The default for OSX is to include the all the notifications in the notification center, which is a lot. This is easily configured, but it can only done after the fact (as far as I know)
* It’s another dependency

## Alternatives

### Consider the current error page enough.
This approach may be enough for many cases. It doesn’t require an additional dependency.

### Improve tooling and education around editor linting to catch errors sooner
Catching the problems before you start the build process will always be better better solution. We should continue work towards this. The challenge is this requires a lot more education and setup for new developers. This works against our already steep upfront learning curve. 

### Include a web based flash messaging system for builds
This could work similarly to the `ember-cli-build-notifications`, but exist solely in the browser. It would notify on success/error using flash messaging system that is part of the dev build. 

## Unresolved questions

* Though I'm advocating for all notifications, the default settings are certainly up for discussion
* It strikes me that including a "Build started" notification may be worth adding to the addon
