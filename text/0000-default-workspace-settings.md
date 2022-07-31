---
Stage: Accepted
Start Date:
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s):
RFC PR:
---

<!---
Directions for above:

Stage: Leave as is
Start Date: Fill in with today's date, YYYY-MM-DD
Release Date: Leave as is
Release Versions: Leave as is
Relevant Team(s): Fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies
RFC PR: Fill this in with the URL for the Proposal RFC PR
-->

# Default workspace settings for code editor

## Summary

Adding the default workspace settings will help new developers to start working in Ember project.

## Motivation

For example, what features can give us [Visual Studio Code](https://code.visualstudio.com), which is the [most popular](https://survey.stackoverflow.co/2022/#section-most-popular-technologies-integrated-development-environment) code editor.

### Recommended extensions

```json
// .vscode/extensions.json
{
	"recommendations": [
    "EditorConfig.EditorConfig",
		"esbenp.prettier-vscode",
		"dbaeumer.vscode-eslint",
		"lifeart.vscode-ember-unstable",
		"lifeart.vscode-glimmer-syntax",
		"chiragpat.vscode-glimmer",
	],
}
```

### Settings for to debug inside VS Code

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Launch Chrome",
      "type": "chrome",
      "request": "launch",
      "preLaunchTask": "Ember serve",
      "url": "http://localhost:4200",
      "webRoot": "${workspaceFolder}",
      "sourceMapPathOverrides": {
        "test-app/*": "${workspaceRoot}/app/*",
        "test-app/tests/*": "${workspaceFolder}/tests/*",
      },
    },
  ]
}
```

### Tasks to run the scripts

```json
// .vscode/tasks.json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Ember serve",
      "type": "npm",
      "script": "start",
      "isBackground": true,
      "group": {
        "kind": "build",
        "isDefault": true
      },
    },
  ]
}
```

### Other useful settings

```json
// .vscode/settings.json
{
  "search.exclude": {
    "dist": true
  }
}
```

Another code editors may have different opinions.

## Detailed design

Optional flag `--code-editor=vscode` of the `new` command will add workspace settings in new projects from the blueprint. Usually code editors store the workspace settings in the root, like VS Code uses `.vscode` folder. `--code-editor` flag can have another values for various code editors. If the flag is not settled, cli will do nothing.

## How we teach this

We would need to add new feature to the [CLI commands reference](https://cli.emberjs.com/release/advanced-use/cli-commands-reference/) page.


## Drawbacks

This feature shouldn't touch other existing or planned features or existing apps.

## Alternatives

Doing nothing.

## Unresolved questions

None.
