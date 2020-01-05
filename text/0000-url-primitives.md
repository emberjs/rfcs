- Start Date: 2020-01-04
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/570
- Tracking: (leave this empty)

# URL Manager

Supersedes [Add queryParams to the router service](https://github.com/emberjs/rfcs/pull/380)

## Summary

A URL Manager will provide the ability to manipulate the URL, 
both when writing to `window.location` (serializing), 
and reading from `window.location` (deserializing). 
The ultimate goal of the URL Manager will be to enable 
app and addon authors to customize the use of the URL—
including internationalized routes, dynamic segment slugs / aliases, 
and alternative query params behavior—though, 
implementation of those specific things is outside the scope of *this* RFC. 

## Motivation

There is currently no way to alter the router's behavior around the interpretation of the URL. 
This means that we are unable to provide internationalized URLs, 
we cannot customize the (de)serialization of query params 
(which is important because there is no standard format for query params).

These APIs will also provide an opportunity to iterate on a better default query params implementation that will enable an objectively better development experience for app devs 
when interacting with query params.

If no URL Manager is configured, Ember's current URL behavior will be preserved.

## Examples

The following examples of the proposed API are written in TypeScript in _userland_/_application-space_ to better 
demonstrate intended API and available data.

The same naming semantics and conventions in today's router system will still apply. 

All examples will have the following in common:
```ts
// router.js
import EmberRouter, { URLManager } from '@ember/routing/router';
import config from 'app-name/config/environment';

// ... Example CustomURLManager here ...

export default class Router extends EmberRouter {
  location = config.locationType;
  rootURL = config.rootURL;
  urlManager = CustomURLManager; // Defined in Examples
});

Router.map(function() {
  this.route('blogs', function() {
    // index route would be the "list" of blogs
    this.route('blog', { path: ':id' }, function() {
      this.route('posts', function() {
        // index route would be the "list" of posts
        this.route('post', { path: ':id' });
      })
    });
  })
});
```


### Naïve Query Params without Controllers

This example proposes the possibility of an alternate strategy for managing query params without the need for controllers. 
Note that this would not change or alter the existing query param behavior in any way.

```ts
// app/router.js/ts
import { inject as service } from '@ember/service';
import qs from 'qs';

class CustomURLManager extends URLManager {
  @service router;

  fromURL(url: string): RouteInfo { // /blogs/1/posts/2?foo=bar
    let [path, query] = url.split('?');

    let routeInfo = this.router.recognize(path);
    // Because query params have no standardized way of (de)serialization,
    // there has been no way to transform deep objects or arrays.
    // This gives control over this process, allowing existing code that hacked
    // around this limitation to be deleted.
    let queryParams = qs.parse(query);

    return {
      ...routeInfo,
      queryParams,
    };
  }

  toURL(routeInfo: RouteInfo) {
    let { 
      mapInfo: {
        segments // [blogs, :blogId, posts, :postId]
      },
      queryParams, // { foo: 'bar' }
      dynamicSegments,
    } = routeInfo;

    let url = segments.map(segment => dynamicSegments[segment] || segment).join('/');

    let query = qs.stringify(queryParams);

    return `/${url}?${query}`; // => /blogs/1/posts/2?foo=bar
  }
}
```

### i18n routes
In this scenario, we may want to internationalized route segments.

For example, in English, we may want a route to be `/blogs/1/posts/2`, 
but in Korean, `/블로그/1/게시물/2`.

```ts
// app/router.js/ts
class CustomURLManager extends URLManager {
  // current language: Korean
  @service i18n;
  @service router;

  fromURL(url: string): RouteInfo { // /블로그/1/게시물/2?foo=bar
    let [path, query] = url.split('?');
    let segments = path.split('/');

    // this dosen't have to be english, but it does have to match the names as
    // defined in Router.map(...);
    let english = 
      path.split('/')
          .segments.map(segment => {
            return this.i18n.lookup(`routes.from.${segment}`, 'en-us') || segment;
          })
          .join('/');

    let routeInfo = this.router.recognize(english);
    let queryParams = qs.parse(query);

    return {
      ...routeInfo,
      queryParams,
    };
  }

  toURL(routeInfo: RouteInfo) {
    let { 
      mapInfo: {
        segments // [blogs, :blogId, posts, :postId]
      },
      queryParams, // { foo: 'bar' }
      dynamicSegments,
    } = routeInfo;

    let url = segments.map(segment => {
      return dynamicSegments[segment] || this.i18n.t(`routes.${segment}`);
    }).join('/');

    let query = qs.stringify(queryParams);

    return `/${url}?${query}`; // => /블로그/1/게시물/2?foo=bar
  }
}


```

### Custom URL Management per-route
There are situations in which links and their routes don't conform to the rest of the application. For these situations, here is an example showing how manage the URL separately for those routes.

```ts
// app/router.js/ts
Router.map(function() {
  this.route('no-query-params', {
    fromURL(url: string): RouteInfo {
      let [path, query] = url.split('?');
      
      return this.router.recognize(path);
    },
    toURL(routeInfo: RouteInfo, dynamicSegments: object) {
      return this.router.urlFor({routeInfo, queryParams: undefined });
    }
  });
  // this.route('posts', ...)
});

class CustomURLManager extends URLManager {
  @service router;

  fromURL(url: string): RouteInfo {
    let [path, query] = url.split('?');
    // NOTE: this method could be named anything, as long as it matches
    //       what is in the options / mapInfo object of the Router.map
    let { fromURL } = this.router.mapInfoFrom(path)

    if (fromURL) return fromURL(url);

    // ...
  }

  toURL(routeInfo: RouteInfo, dynamicSegments: object) {
    let { mapInfo: { toURL } } = routeInfo;

    if (toURL) return toURL(routeInfo, dynamicSegments);

    // ...
  }
}
```

## Detailed design

### Additions to existing Public APIs
1. add full list of dynamicSegments to RouteInfo so that the task of building out the map of dynamic segments to their values isn't in user-space
2. Restriction of Query Params on controllers / `<LinkTo />` needs to have a way to opt-out. 
   Today, if a query param is added to a `<LinkTo />` and that query param is not present on the target `route`'s controller, the query param is removed from the link. 
   Query Param allow/deny lists could be re-implemented using the Router `MapInfo` / options.
3. The router's urlFor helper function should be able to take a RouteInfo / RouteInfo should be able to be converted to an URL
   - delegates to `urlManager.toURL` for `RouteInfo`
4. Static `MapInfo` object reference added to each `RouteInfo`.
5. routerService.mapInfoFrom should take: path / url / routeInfo -- uses existing recognize method


### Changes to Internals

1. `<LinkTo />` and other related router helpers use `toURL` and `fromURL` from the URL Manager.
   This allows more control over the allow/deny list behavior of query params filtering.

### Additional APIs / Behavior

1. URLManager is a container-controlled object, so that it may utilize the dependency injection system.
  Primary need for this is to access the router service to utilize existing APIs for transforming URLs and `RouteInfo`s

### Notes

1. `MapInfo` is static or "frozen" -- it is only constructed at the time of router setup.
2. `MapInfo` represents the optional `object` argument of `Router.map`'s `this.route`. Empty object if not configured.
3. If needed, the URL Manager may be used from a component: 
    ```ts
    class Post extends Component {
      @service router;

      get urlForCurrentRoute_butTheLongWay() {
        // the same as this.router.currentURL (if currentURL also had queryParams)
        return this.router.urlManager.toURL(this.router.routeInfo);
      }
    }
    ```
4. `transitionTo` and `replaceWith` APIs are unaffected.

### The Default URL Manager

Once implemented, the URL behavior, by default, will function as before the URL Manager -- in that the standard router.js script:

```ts
// router.js
import EmberRouter from '@ember/routing/router';
import config from 'app-name/config/environment';

export default class Router extends EmberRouter {
  location = config.locationType;
  rootURL = config.rootURL;
});

Router.map(...);
```

is sufficient for maintaining backwards compatibility.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

Like the component-manager, and modifier-manager, 
this should be considered a low-level API that most people shouldn't need to interact with.
Addon authors may implement different URL-handling techniques and export an URL manager 
for app-devs to assign in the router.js file.

Maybe after a bit of exploration (maybe of query params, specifically), a particular approach may be pulled in to Ember.

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

The guides don't need any changes, but the API documentation would need to be thorough.

## Drawbacks

The biggest drawback is that it would be easy to break routing in the app.
During implementation, this could be mitigated by providing a generated route unit test that 
symmetrically checks that toURL and fromURL are inverses of each other for the given route's expected URL / routeInfo.

```ts
import { module, test } from 'qunit';
import { setupTest } from 'ember-qunit';

module('Unit | Route | blogs/blog/posts/post', function(hooks) {
  setupTest(hooks);

  test('it exists', function(assert) {
    let route = this.owner.lookup('route:blogs/blog/posts/post');
    assert.ok(route);
  });

  test('urls are resolved', function(assert) {
    let router = ;
    let sampleUrl = '/blogs/1/posts/2';
    let resultUrl = toURL(fromURL(sampleURL))

    assert.equal(resultUrl, sampleUrl);

    let mapInfo = router.mapInfoFor('blogs.blog.posts.post');

    let sampleRouteInfo = {
      mapInfo,
      queryParams: {}
      dynamicSegments: { blog: '1', post: '2' },
    }
    let resultRouteInfo = fromURL(toURL(sampleRouteInfo));

    assert.deepEqual(resultRouteInfo, sampleRouteInfo);
  });
});
```

## Alternatives

- Builtin regex matchers for the routes
  - hard to debug
  - can often match on incorrect portions of the Route if not thoroughly tested

- Only adding QueryParams modifications (per RFC #380)
  - not flexible enough
  - forces a single query params implementation

Prior Art: 
- the manager patterns from elsewhere in Ember.

## Unresolved questions

- Should the URL Manager exist as a static config or an instantiable object per-route? The above proposal is a static config / singleton, but allowing an instance per route would allow for more varied state buckets, but could also make debugging harder as there would be an URL Manager for each route segment.

- `toURL` / `fromURL` or `serialize` / `deserialize`?
