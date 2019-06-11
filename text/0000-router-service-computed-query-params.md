- Start Date: 2018-09-21
- RFC PR: [https://github.com/emberjs/rfcs/pull/380](https://github.com/emberjs/rfcs/pull/380)
- Ember Issue: (leave this empty)

# Add `@queryParam` decorator and `QueryParamsService`

## Summary

Access to query params is currently restricted to the controller, and subsequently, the corresponding route. 
This results in some limitations with which users may consume query param data. 
By exposing query params on a new `QueryParamsService`, users will be able to easily access the params from deep down a component tree, removing the need to pass query-param related data and actions down many layers of components.

## Motivation

Modern SPA concepts have converged on the idea that query params should be easily accessible from the object responsible for handling the route.
Like with the [RouterService](https://github.com/emberjs/rfcs/blob/master/text/0095-router-service.md), 
it is common to have a need to perform routing behavior from deep down a component tree.
Additionally, the current query params implementation feels very verbose for "just wanting to access a property". Accessing data within the url **should feel easy**.

### Examples

An addon has been written to demonstrate usage: https://github.com/NullVoxPopuli/ember-query-params-service

Example usage:
```ts
import Component from "@glimmer/component";
import { queryParam } from "ember-query-params-service";

export default class SomeComponent extends Component {
  @queryParam foo;

   addToFoo() {
    this.foo = (this.foo || 0) + 1;
  }
}
```

Having query params accessible on the router service would allow users to implement:

 - query param aware modals that may hide or show depending on the presence of a query param.
 - fill in form fields from a link.
 - filter / search components could update the query param property.
 - whatever else query params are used for outside of a SPA.

## Detailed design

Add a service that maintains properties derived from a split of `window.location.search` into objects, which could have deeply nested objects and arrays.
The properties must maintain state based on the current route, which mimics the current behavior that the singleton controller's long-lived state provide with respect to maintaining query-param values.
Ensure that setting any deeply nested value in the query params object computed property also updates the URL.
A `queryParam` decorator for accessing and updating query params in the URL must exist for short-hand use in components, routes, etc.

The biggest change needed, which could potentially be a breaking change, is that the allow-list on routes' queryParams hash will need to be removed as people transition to this form of queryParams management. The controller-based way is static in that all known query params must be specified on the controller ahead of time. This has been a great deal of frustration amongst many developers, both new and old to ember.
This is a dynamic way to manage query params which hopefully aligns more with people's mental model of query params. 

Additionally, in order to be backwards-compatible with `{ refreshModel: true }`, the model hook will need to re-run if any tracked properties used within the model hook are changed. 

Example implementation of the service:
```ts
import Service, { inject as service } from '@ember/service';
import RouterService from '@ember/routing/router-service';

import { tracked } from '@glimmer/tracking';
import * as qs from 'qs';

interface QueryParams {
  [key: string]: number | string | undefined | QueryParams;
}

interface QueryParamsByPath {
  [key: string]: QueryParams;
}

export default class QueryParamsService extends Service {
  @service router!: RouterService;

  @tracked current!: QueryParams;
  @tracked byPath: QueryParamsByPath = {};

  constructor(...args: any[]) {
    super(...args);

    this.setupProxies();
  }

  init() {
    super.init();

    this.updateParams();

    this.router.on('routeDidChange', () => this.updateParams());
    this.router.on('routeWillChange', transition => this.updateURL(transition));
  }

  get pathParts() {
    const [path, params] = (this.router.currentURL || '').split('?');
    const absolutePath = path.startsWith('/') ? path : `/${path}`;

    return [absolutePath, params];
  }

  private setupProxies() {
    let [path] = this.pathParts;

    this.byPath[path] = this.byPath[path] || {};

    this.current = new Proxy(this.byPath[path], queryParamHandler);
  }

  private updateParams() {
    this.setupProxies();

    const [path, params] = this.pathParts;
    const queryParams = params && qs.parse(params);

    this.setOnPath(path, queryParams);
  }

  /**
   * When we have stored query params for a route
   * throw them on the URL
   *
   */
  private updateURL(transition: any /* Transition doesn't have intent.. some how? */) {
    const path = transition.intent.url;
    const { protocol, host, pathname, search } = window.location;
    const queryParams = this.byPath[path];
    const existing = qs.parse(search.split('?')[1]);
    const query = qs.stringify({ ...existing, ...queryParams });
    const newUrl = `${protocol}//${host}${pathname}?${query}`;

    window.history.replaceState({ path: newUrl }, '', newUrl);
  }

  private setOnPath(path: string, queryParams: object) {
    this.byPath[path] = this.byPath[path] || {};

    Object.keys(queryParams || {}).forEach(key => {
      let value = queryParams[key];
      let currentValue = this.byPath[path][key];

      if (currentValue === value) {
        return;
      }

      if (value === undefined) {
        delete this.byPath[path][key];
        return;
      }

      this.byPath[path][key] = value;
    });
  }
}

const queryParamHandler = {
  get(obj: any, key: string, ...rest: any[]) {
    return Reflect.get(obj, key, ...rest);
  },
  set(obj: any, key: string, value: any, ...rest: any[]) {
    let { protocol, host, pathname } = window.location;
    let query = qs.stringify({ ...obj, [key]: value });
    let newUrl = `${protocol}//${host}${pathname}?${query}`;

    window.history.pushState({ path: newUrl }, '', newUrl);

    return Reflect.set(obj, key, value, ...rest);
  },
};

```

Example implementation of the decorator:

```ts
import { get, set } from '@ember/object';
import { getOwner } from '@ember/application';
import { tracked } from '@glimmer/tracking';
import { default as QueryParamsService } from '../services/query-params';

export interface ITransformOptions<T> {
  deserialize?: (queryParam: string) => T;
  serialize?: (queryParam: T) => string;
}

type Args<T> = [] | [string, ITransformOptions<T>] | [ITransformOptions<T>] | [string];

export function queryParam<T = boolean>(...args: Args<T>) {
  return (target: any, propertyKey: string, sourceDescriptor?: any) => {
    const { set: oldSet } = tracked(target, propertyKey, sourceDescriptor);
    const [propertyPath, options] = extractArgs<T>(args, propertyKey);

    const result = {
      configurable: true,
      enumerable: true,
      get: function(): T {
        // setupController(this, 'application');
        const service = ensureService(this);
        const value = get<any, any>(service, propertyPath);
        const deserialized = tryDeserialize(value, options);

        return deserialized;
      },
      set: function(value: any) {
        // setupController(this, 'application');
        const service = ensureService(this);
        const serialized = trySerialize(value, options);

        set<any, any>(service, propertyPath, serialized);
        oldSet!.call(this, serialized);
      },
    };

    return result as any;
  };
}

function extractArgs<T>(args: Args<T>, propertyKey: string): [string, ITransformOptions<T>] {
  const [maybePathMaybeOptions, maybeOptions] = args;

  let propertyPath: string;
  let options: ITransformOptions<T>;

  if (typeof maybePathMaybeOptions === 'string') {
    propertyPath = `current.${maybePathMaybeOptions}`;
    options = maybeOptions || {};
  } else {
    propertyPath = `current.${propertyKey}`;
    options = maybePathMaybeOptions || {};
  }

  return [propertyPath, options];
}

function tryDeserialize<T>(value: any, options: ITransformOptions<T>) {
  if (!options.deserialize) return value;

  return options.deserialize(value);
}

function trySerialize<T>(value: any, options: ITransformOptions<T>) {
  if (!options.serialize) return value;

  return options.serialize(value);
}

// could there ever be a problem with using only one variable in module-space?
let qpService: QueryParamsService;
function ensureService(context: any): QueryParamsService {
  if (qpService) {
    return qpService;
  }

  qpService = getQPService(context);

  return qpService;
}

function getQPService(context: any) {
  return getOwner(context).lookup('service:queryParams');
}
```

## How we teach this

Currently, query params _must_ be [specified on the controller](https://guides.emberjs.com/release/routing/query-params/):
```ts
export default class extends Controller {
  queryParams = ['page', 'filter', {
   // QP 'articles_category' is mapped to 'category' in our route and controller
   category: 'articles_category'
  }];
  category = null;
  page = 1;
  filter = 'recent';
  
  @computed('category', 'model')
  get filteredArticles() {
    // do something with category and model as category changes
  }
}
```
The default happy path should be one  that suggests usage of the `@queryParam` decorator. This is the most flexible, and prevides serialization/deserialization options per-query param if needed. Maybe using [@pzuraq's macro-decorators](https://pzuraq.github.io/macro-decorators/), users could wrap `@queryParam` for different data types.

```ts
import Component from "@glimmer/component";
import { queryParam } from "ember-query-params-service";

export default class SomeComponent extends Component {
  @queryParam foo;

   addToFoo() {
    this.foo = (this.foo || 0) + 1;
  }
}
```


Having computed properties available elsewhere will be a shift in thinking that "the controller manages query params" to "the service that allows access to the query params manages the query params"

```ts
import Route from '@ember/routing/route';
import { inject as service } from '@ember/service';
import { alias } from 'ember-query-params-service';

export default class ApplicationRoute extends Route {
  @service queryParams;

  @alias('queryParams.current.r') isSpeakerNotes;
  @alias('queryParams.current.slide') slideNumber;

  model() {
    return {
      isSpeakerNotes: this.isSpeakerNotes,
      slideNumber: this.slideNumber
    }
  }
}
```

## Drawbacks

- This requires tracked properties as the tracking system is perfect for propagating updates whenever the internal queryParam tracking object changes -- in the QueryParamsService's `byPath` and `current` properties. So only the most up-to-date code-bases would be able to benefit.
- The old behavior of controller-based query params where query params live on the controller that is a singleton and are restored whenever a route is returned to is still possible / implemented with this QueryParamService, but because components can modify query params at any depth, this may introduce hard-to-trace behaviors.
- Some people may be relying on the controller query-params allow-list.

## Alternatives

React Router, for example, includes the query params as a string with every Route component via the `location` object.
They also provide url segments, which may also be handy, but for now, may be outside the scope of this RFC.
