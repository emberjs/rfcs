* Start Date: 2017-12-17
* RFC PR: (leave this empty)
* Ember Issue: (leave this empty)

# Node Modules

## Summary

Introduce the ability to consume packages from node_modules, without wrapping it in a plug-in.

```bash
npm install cool-thing
```

```javascript
import coolThing from 'cool-thing';
```

## Motivation

Modern JavaScript projects typically allow this form of use from common js modules from npm. This would make Ember/Ember-CLI feel more at home with how the rest of the community operates.

## Detailed design

(assume yarn works as well)

We should look at other bundlers that offer this feature, and copy the best UX.

For example `webpack` will allow you to consume an CJS module in a couple different ways.

How webpack handles CJS:

Tests were done here: https://github.com/rtorr/webpack-modules-test

```javascript
// common-js
require('cool-thing');

// es-modules
import 'cool-thing';
```

```javascript
// foo.js
exports.foo = function() {
  return 'foo';
};

// main.js
import './foo'; // {foo: ƒ}
require('./foo'); // {foo: ƒ}

import foo from './foo';

console.log(foo); // {foo: ƒ}
```

```javascript
// foo.js

module.exports = {
  test: 'test'
};

exports.foo = function() {
  return 'foo';
};

// main.js
import './foo'; // {test: 'test'}
require('./foo'); // {test: 'test'}

import foo from './foo';

console.log(foo); // {test: 'test'}
```

```javascript
// foo.js

exports.foo = function() {
  return 'foo';
};

// main.js
import { foo } from './foo';

console.log(foo); // ƒ () {return 'foo';}
```

## How We Teach This

I think we would lean on the work other tools have done here, and try and copy those learning materials.

## Drawbacks

We might have to design and teach two things. Ember addons vs node_modules.

## Alternatives

There are a few alternative module consumption designs:

* Typescript
* Browserify
* Rollup

## Unresolved questions

Teaching
What to do about ember add-ons
