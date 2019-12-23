- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# URL Manager

Supersedes [Add queryParams to the router service](https://github.com/emberjs/rfcs/pull/380)

## Summary

An URL Manager will provide the ability to manipulate the URL -- 
both when writing to the `window.location` (serializing), 
and reading from the `window.location` (deserializing). 
The ultimate goal of the URL Manager will be to enable 
app and addon authors to customize the use of the URL -- 
including i18n'd routes, dynamic segment slugs / aliases, 
and alternate query params behavior -- though, 
implementation of those specific things is outside the scope of *this* RFC. 

## Motivation

There is currently no way to alter the router's behavior around the interpretation of the URL. 
This means that we are unable to provide internationalized URLs, 
we cannot customize the (de)serialization of query params 
(which is important because there is no standard format for query params).

These APIs will also provide an opportunity to iterate on a better default query params implementation that will enable an objectively better development experience for app devs 
when interacting with query params.

The current URL behavior will be preserved, should the URL Manager be implemented. 

## Detailed design

The implementation of the URL Manager is 2 parts:
 - the URL Manager itself
 - access to the URL Manager / Router.map data from framework-objects
   - required for preserving existing query params behavior where query params are, by default, allow-list only.


### Examples

Custom URL Manager that implements i18n routes
```ts
// app/router.js/ts
import EmberRouter, { URLManager } from '@ember/routing/router';
import { inject as service } from '@ember/service';
import config from 'app-name/config/environment';
import qs from 'qs';


class CustomURLManager extends URLManager {
  // current language: Korean
  @service i18n;
  @service router;

  // interprets the URL
  deserialize(url: string): RouteInfo { // /블로그/1/게시물/2?foo=bar
    let [path, query] = url.split('?');
    let segments = path.split('/');
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

  // pushes the state to the URL
  serialize(routeInfo: RouteInfo, dynamicSegments: object) {
    let { 
        mapInfo: {
            segments // [blogs, :blogId, posts, :postId]
        }, // MapInfo
        queryParams, // { foo: 'bar' }
        name, // blog.posts
    } = routeInfo;

    let url = segments.map(segment => {
      return dynamicSegments[segment] || this.i18n.t(`routes.${segment}`);
    }).join('/');

    let query = qs.stringify(queryParams);

    return `/${url}?${query}`; // => /블로그/1/게시물/2?foo=bar
  }
}

export default class Router extends EmberRouter {
  location = config.locationType;
  rootURL = config.rootURL;
  urlManager = CustomURLManager;
});

Router.map(function() {

});
```

**Accessing Router Map Info**
[RouteInfo](https://api.emberjs.com/ember/release/classes/RouteInfo)
 - added property: `mapInfo`:
    ```ts
    interface MapInfo {
      name: string;
      segments: string[]; // [blogs, :blogId, posts, :postId]
      options: {
        path: string;

      };
      childRoutes: MapInfo[]
    }
    ```


```ts
class Post extends Component {
  @service router;

  get dynamicSegments() {
      this.router.routeInfo.mapInfo.options;
  }
}
```

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
