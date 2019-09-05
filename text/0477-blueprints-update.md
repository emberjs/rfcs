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

The code is already completed [here](https://github.com/ember-cli/ember-cli-update/pull/552), but I wanted to publish this RFC before releasing it into the wild. I want to make sure the format of these files are agreed upon, because if this is successful, it will probably propagate throughout the ecosystem.

```json
// config/ember-cli-update.json
{
  "blueprints": [
    {
      "name": "my-custom-blueprint",
      "version": "0.0.1"
    }
  ]
}
```

`ember-cli-update.json` will be rewritten by the tool, which is why JSON is preferred over JS.

How should the state be structured?

```json
{
  "blueprints": [
    {
      "name": "my-custom-blueprint",
      "version": "0.0.1"
    }
  ]
}
```

```json
{
  "blueprints": {
    "my-custom-blueprint": {
      "version": "0.0.1"
    }
  }
}
```

Future-proofing for more options

```json
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

**Commands**

***bootstrap***

```
ember-cli-update bootstrap
```

This starts an `ember-cli-update.json` file for you with the detected ember-cli blueprint version. It uses the same logic we currently have for detecting the ember-cli blueprint version, but now since the state is stored, we don't have to run the detection logic anymore.

***init***

```
ember-cli-update init
ember-cli-update init -b libkit
ember-cli-update init -b my-custom-blueprint --custom-option-1 --custom-option-2 foo
```

This is similar to `ember init` or `ember-cli-update --reset`, but also stores the blueprint information in `ember-cli-update.json`. It also stores any options used to create future update diffs.

***install***

```
ember-cli-update install ember-cli-mirage
```

This is the same as `ember install ember-cli-mirage` except that it stores the blueprint state for later updating.

***save***

```
ember-cli-update save libkit --from 0.5.18
ember-cli-update save ember-cli-mirage --from 0.4.0
```

This is a helper for saving the state of an old blueprint's run without ember-cli-update. The usefulness of this is questionable, because you have to remember something you did in the past. Its only usefulness is getting a project set up without having to know the `ember-cli-update.json` schema.

**Methods of delivery**

Blueprints could be responsible for injecting the state into projects

```json
// my-custom-blueprint/blueprints/my-custom-blueprint/files/config/ember-cli-update.json
{
  "blueprints": [
    {
      "name": "my-custom-blueprint",
      "version": "<%= blueprintVersion %>"
    }
  ]
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

**Preserving options**

I'm not sure if anyone does it now, but it could be possible to handle options when generating a new project via a custom blueprints.

```
ember new my-app -b my-blueprint --option1 --option2=foo
```

```json
// my-blueprint/files/config/ember-cli-update.json
{
  "blueprints": [
    {
      "name": "my-blueprint",
      "version": "<%= blueprintVersion %>",
      "options": [
        "option0=<%= option0 %>",
        "option1=<%= option1 %>",
        "option2=\"<%= option2 %>\""
      ]
    }
  ]
}
```

```json
// my-app/config/ember-cli-update.json
{
  "blueprints": {
    "my-blueprint": {
      "version": "0.0.1",
      "options": [
        "option0=false",
        "option1=true",
        "option2=\"foo\""
      ]
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

```json
{
  "blueprints": {
    "my-blueprint": {
      "version": "0.0.1",
      "codemods": "some-server.com/my-blueprint-codemods-manifest.json"
    }
  }
}
```

TODO: The options need to be normalized somehow to be used in the codemods system.

`--custom-option foo` vs `--custom-option=foo` vs `--custom-option="foo"`

```js
// This would probably be bad as `foo` would be treated as an option
"options": [
  "--custom-option",
  "foo"
]
// This is better
"options": [
  "--custom-option=\"foo\""
]
// or maybe this?
"options": [
  "customOption=\"foo\""
]
```

## How we teach this

I’m not sure. It could be a section in the ember-cli-update README, or leave it up to the blueprints that want to support this to document. If this feature takes off in the ecosystem, then it might warrant a guides section on keeping your blueprints up-to-date.

## Drawbacks

A drawback could be that blueprints start writing this state, but the consumer doesn’t want to use ember-cli-update to keep it in sync. In that case, it’s another file the user has to delete.
