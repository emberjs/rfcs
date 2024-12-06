---
stage: accepted
start-date: 2023-12-26T00:00:00.000Z
release-date:
release-versions:
teams: # delete teams that aren't relevant
  - framework
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1001
project-link:
---

<!---
Directions for above:

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
-->

# Deprecate named `inject` export from `@ember/service`

## Summary

As of [`ember-source@4.1`](https://blog.emberjs.com/ember-4-1-released) (and [RFC#752](https://github.com/emberjs/rfcs/pull/752)),  `inject` is an old alias that's no longer needed

## Motivation

`import { service } from '@ember/service'`
makes more sense than 
`import { inject as service } from '@ember/service'`

This allows us to slim down our public API surface area to more of _what's needed_.


## Transition Path

Most folks can do a mass find and replace switch from `inject as service` to just `service`.

An example codemod could look [something like this](https://astexplorer.net/#/gist/119f88339ea024e7cde63c71f52ce216/4d128a1239cbb56e00a69d3f710d67c20ed0e431)
```js 
export const parser = 'ts'

export default function transformer(file, api) {
  const j = api.jscodeshift;
  
  const importNames = new Set();

  const root = j(file.source);
  
  // find things we want to get rid of
  root
    .find(j.ImportSpecifier)
    .forEach(path => {
      if (path.node.imported.name === 'inject') {
      	importNames.add(path.node.local.name);
      }
    })
  
  // now it's time to replace
  root.find(j.ClassProperty).forEach(path => {
    let node = path.node;
    
    let hasInject = hasDecorators(node, [...importNames.values()]);
    
    if (!hasInject) return;
    
    node.decorators = node.decorators.map(decorator => {
      let { expression } = decorator;
      
      if (expression.type === 'Identifier') {
        if (importNames.has(expression.name)) {
          decorator.expression = j.identifier('service');
        }
      }
      
      if (expression.type === 'CallExpression') {
        decorator.expression.callee = j.identifier('service');
      }
      
      return decorator;
    });
  });
  
  return root.toSource();
}

// Copied from: https://github.com/NullVoxPopuli/ember-concurrency-codemods/tree/main
function firstMatchingDecorator(node, named = []) {
  if (!node.decorators) return;

  return node.decorators.find((decorator) => {
    let { expression } = decorator;

    switch (expression.type) {
      case 'MethodDefinition': {
      }
      case 'CallExpression': {
        let { callee } = expression;

        switch (callee.type) {
          case 'Identifier':
            return named.includes(callee.name);
          case 'MemberExpression': {
            let { object } = callee;

            return named.includes(object.callee.name);
          }
        }
      }
      case 'Identifier':
        return named.includes(expression.name);
    }
  });
}

function hasDecorators(node, named = []) {
  return Boolean(firstMatchingDecorator(node, named));
}
```

<details><summary>The test scenarios</summary>

```ts 
import { inject } from '@ember/service';
import { inject as service } from '@ember/service';
// import Service from '@ember/service';
import BaseService from '@ember/service';
import { inject as serviceDecorator } from '@ember/service';
import { inject as x } from '@ember/service';
// import { service } from '@ember/service';
import { service as y } from '@ember/service';
// import Service, { inject, service } from '@ember/service';
import Service, { inject as s } from '@ember/service';


export default class Demo extends Service {
  
}

export default class Demo2 extends BaseService {
  // simple
  @inject router;
  @service router1;
  @x router2;
  @y router3;
  @serviceDecorator router4;
  @inject('router') router41;
  
  // TS-only
  @inject declare router5: Type;
  @inject('router') declare router51: Type;
  @service declare router6: Type;
  @x declare router7: Type;
  @y declare router8: Type;
  @serviceDecorator declare router9: Type;
}
```

</detailS>


## How We Teach This

The docs / guides already use the new import path.

## Drawbacks

As with any deprecation, we introduce an upgrade cliff for addons that are updated infrequently, and consequently their consuming apps.
As a mitigation, we could, for v1 addons, add an additional transform to ember-cli-babel to automatically upgrade `inject` from `@ember/service` to `service`.
This does narrow the range a bit, as `service` was introduced in ember-source@4.1, so libraries could not support from 3.28 to 6 (or whichever major ends up removing the `inject`) without adding `@embroider/macros` to conditionally import `inject` or `service` based on the consumer's ember-source version.

## Alternatives

do nothing, the cost of an export alias is:
- a few extra bytes
- mental gymnastics for teaching
- "another case to cover" for tooling

add a lint against `inject`
- all the downsides of the above ("do nothing") may still be present

## Unresolved questions

n/a
