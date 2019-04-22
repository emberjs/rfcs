- Start Date: 2019-04-17
- Relevant Team(s): Ember CLI
- RFC PR: https://github.com/emberjs/rfcs/pull/477
- Tracking: (leave this empty)

# Syncing Blueprints

## Summary

I want to use [ember-cli-update](https://github.com/ember-cli/ember-cli-update) to keep any blueprint up-to-date. Since ember-cli-update can not possibly contain knowledge of all blueprints, some state must be stored in a project.

## Motivation

Projects like [libkit](https://github.com/tildeio/libkit), [@glimmer/blueprint](https://github.com/glimmerjs/glimmer.js/tree/master/packages/%40glimmer/blueprint), [@ember/octane-app-blueprint](https://github.com/ember-cli/ember-octane-blueprint), and other app replacement blueprints become out-of-date instantly after generation. Tweaks to the blueprints are only beneficial to newly generated apps.

## Detailed design

**State storage schema**

Up until now, ember-cli-update could update apps without any configuration, but in order to handle all blueprints and their options, some state has to be stored in the project.

The code is already completed https://github.com/ember-cli/ember-cli-update/pull/552, but I wanted to publish this RFC before releasing it into the wild. I want to make sure the format of these files are agreed upon, because if this is successful, it will probably propagate throughout the ecosystem.

Where should the state go?

```js
// .ember-cli
{
  "ember-cli-update": {
    "blueprints": [
      {
        "name": "my-custom-blueprint",
        "version": "0.0.1"
      }
    ]
  }
}
```

```js
// ember-cli-update.json
{
  "blueprints": [
    {
      "name": "my-custom-blueprint",
      "version": "0.0.1"
    }
  ]
}
```

```js
// package.json
{
  // ...
  "ember-cli-update": {
    "blueprints": [
      {
        "name": "my-custom-blueprint",
        "version": "0.0.1"
      }
    ]
  }
}
```

I vote “ember-cli-update.json”. Since the code will be modified by the updater, it’s easier to reformat JSON. “.ember-cli” I think supports JS, so it would be hard to modify. “package.json” seems like an abuse to put more metadata in there.

I've found via further testing that mixing a file that is tracked by the default ember-cli blueprint (.ember-cli, package.json) and modified by ember-cli-update (to update the blueprint metadata), it gets cumbersome. Both processes try to edit the same file. So in theory, ember-cli could alter the .ember-cli file in a way that conflicts with what you have in yours. This would give you a git conflict, then the updater can't read it anymore because it is in a state of invalid JS. I think this further supports a separate file.

How should the state be structured?

```js
{
  "blueprints": [
    {
      "name": "my-custom-blueprint",
      "version": "0.0.1"
    }
  ]
}
```

```js
{
  "blueprints": {
    "my-custom-blueprint": {
      "version": "0.0.1"
    }
  }
}
```

Future-proofing for more options

```js
{
  "additionalFutureOptions": null,
  "blueprints": {
    "my-custom-blueprint": {
      "version": "0.0.1",
      "additionalFutureOptions": null,
    }
  }
}
```

**Methods of delivery**

Blueprints could be responsible for injecting the state into projects

```js
// my-custom-blueprint/blueprints/my-custom-blueprint/files/.ember-cli
{
  "ember-cli-update": {
    "blueprints": [
      {
        "name": "my-custom-blueprint",
        "version": "<%= blueprintVersion %>"
      }
    ]
  }
}
```

or ember-cli-update could manage it itself if the blueprint does not provide this info. You would have to provide the initial blueprint version to start from.

The goal is for when you type `ember-cli-update` in your project, you will get a prompt like

```
$ ember-cli-update
Multiple blueprint updates found, which would you like to update?
 * ember-cli
 * ember-cli-mirage
 * ember-ci
```

The reason this is so powerful is any org could create their own _partial_ project blueprint (public or private). This blueprint can make any slight (or massive) tweaks to the official blueprints, and ember-cli-update can keep any project in sync with both the official blueprint and your org's blueprints, at the same time.

**Complete vs partial**

We need a way to denote a project replacement blueprint and a supplemental blueprint. A project replacement removes any sort of default ember-cli blueprint tracking, even if it still looks like a normal Ember project (has ember-cli, ember-addon, etc.). A supplemental blueprint is one that piggy-backs on a complete blueprint and makes certain tweaks to files. Think of ember-cli-mirage's extra project configs, or your company's custom settings on top of a normal Ember app.

```js
{
  "blueprints": {
    "my-app-blueprint": {
      "version": "0.0.1"
    },
    "my-addon-blueprint": {
      "version": "0.0.1",
      "isPartial": true
    }
  }
}
```

```
$ ember-cli-update
Multiple blueprint updates found, which would you like to update?
 * my-app-blueprint
 * my-addon-blueprint
```

```js
{
  "blueprints": {
    "my-app-blueprint": {
      "version": "0.0.1",
      "isPartial": true
    },
    "my-addon-blueprint": {
      "version": "0.0.1",
      "isPartial": true
    }
  }
}
```

```
$ ember-cli-update
Multiple blueprint updates found, which would you like to update?
 * ember-cli
 * my-app-blueprint
 * my-addon-blueprint
```

**Preserving options**

I'm not sure if anyone does it now, but it could be possible to handle options when generating a new project via a custom blueprints.

```
ember new my-app -b my-blueprint --option1 --option2=foo
```

```js
// my-blueprint/files/ember-cli-update.json
{
  "blueprints": [
    {
      "name": "my-blueprint",
      "version": "<%= blueprintVersion %>",
      "options": ["option0=<%= option0 %>", "option1=<%= option1 %>", "option2=\"<%= option2 %>\""]
    }
  ]
}
```

```js
// my-app/ember-cli-update.json
{
  "blueprints": {
    "my-blueprint": {
      "version": "0.0.1",
      "options": ["option0=false", "option1=true", "option2=\"foo\""]
    }
  }
}
```

The updater can now generate a project with the correct options.

```
ember new my-app -b my-blueprint --option0=false --option1=true --option2="foo"
```

**Codemods**

There is no reason why you couldn't provide your own codemods with this system. This would be especially useful for project replacement blueprints. We would use the existing codemod system with version detection, option detection, etc.

```js
{
  "blueprints": {
    "my-blueprint": {
      "version": "0.0.1",
      "codemods": "some-server.com/my-blueprint-codemods-manifest.json"
    }
  }
}
```

## How we teach this

I’m not sure. It could be a section in the ember-cli-update README, or leave it up to the blueprints that want to support this to document. If this feature takes off in the ecosystem, then it might warrant a guides section on keeping your blueprints up-to-date.

## Drawbacks

A drawback could be that blueprints start writing this state, but the consumer doesn’t want to use ember-cli-update to keep it in sync. In that case, it’s another file the user has to delete.
