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

```
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

```
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

```
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

How should the state be structured?

```
{
  "blueprints": [
    {
      "name": "my-custom-blueprint",
      "version": "0.0.1"
    }
  ]
}
```

```
{
  "blueprints": {
    "my-custom-blueprint": {
      "version": "0.0.1"
  }
}
```

Future-proofing for more options

```
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

```
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

The reason this is so powerful is any org could create their own _partial_ app blueprint (public or private). This blueprint can make any slight (or massive) tweaks to the official blueprints, and ember-cli-update can keep any app in sync with both the official blueprint and you'r org's blueprints, at the same time.

## How we teach this

I’m not sure. It could be a section in the ember-cli-update README, or leave it up to the blueprints that want to support this to document. If this feature takes off in the ecosystem, then it might warrant a guides section on keeping your blueprints up-to-date.

## Drawbacks

A drawback could be that blueprints start writing this state, but the consumer doesn’t want to use ember-cli-update to keep it in sync. In that case, it’s another file the user has to delete.
