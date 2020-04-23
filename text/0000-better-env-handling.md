- Start Date: 2020-04-23
- Relevant Team(s): Ember.js, Learning
- RFC PR: [#619](https://github.com/emberjs/rfcs/pull/619)
- Tracking: (leave this empty)

# Better environment handling

## Summary

This RFC proposes to add a tiny module similar to `@glimmer/env`, to allow to more easily work with common environment questions.

## Motivation

This RFC tries to solve two common problems:

1. Handling test-specific code. The old way of doing this, `Ember.testing`, is not really ideal and recommended anymore.
2. Handling debug-specific (=non-production) code. You can use `import { DEBUG } from "@glimmer/env"` for this, or `import { runInDebug } from "@ember/debug"`, although the later [seems to be buggy as of now](https://github.com/emberjs/ember.js/issues/18912).

This RFC proposes to provide a more more streamlined, approved and documented way to handle these cases, by introducing an `@ember/env` namespace which can be used like this:

```js
import { DEBUG, TESTING } from "@ember/env";
```

## Detailed design

```js
import { DEBUG } from "@ember/env";
```

This works exactly as `import { DEBUG } from "@glimmer/env"`.
The reasoning for moving this into the `@ember` namespace is that it is not clear why this should be imported from glimmer, which is the rendering engine.

```js
import { TESTING } from "@ember/env";
```

This works like `Ember.testing`.

Together with this, we should deprecate (or plan to deprecate) the older functions:

- `Ember.testing`
- `runInDebug` from `@ember/debug`
- `DEBUG` from `@glimmer/env`

## How we teach this

This probably doesn't need to be in the guides, just in the API docs for developers looking for this.

## Drawbacks

It slightly extends API surface.

## Alternatives

We could also add `@glimmer/env` to the Ember API docs, like `@glimmer/component` or `@glimmer/tracking`. However, in contrast to them it probably makes less sense to find this under the `@glimmer` namespace.

We could add something like `TESTING` to `@glimmer/env`. The same concerns as mentioned above apply here too.

We could also do nothing and propose users to continue using `Ember.testing` or a home-grown solution instead. However, I think that these problems are common enough to warrant a nice, built-in solution for them.

We could also use a different module name, or put it into an existing module.

## Unresolved questions

Should we deprecate `import { DEBUG } from '@glimmer/env';`, or keep both?

Should we deprecate the older functionality immediately or leave a grace period?
