---
stage: accepted
start-date: 2023-12-15T00:00:00.000Z
release-date:
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - framework
  - learning
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/995
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

# Deprecate non-co-located components.

## Summary

Deprecates
- classic component layout
- pods component layout


After "ember@6", only the following will be allowed:
- co-located components 
- gjs / gts components

## Motivation

These older component layouts force build tooling to keep a lot of resolution rules around, and makes it hard for codemods and other community tooling to effectively work across folks' projects.


## Transition Path

There are two types of paths to migrate off the old layouts 
- use a currently supported multi-file layout (keeping separate `js`, `ts`, and `hbs` files)
- migrate the component entirely to the latest component format, `gjs`, `gts`, (aka `<template>`)

There are some tools to help with this:
- [Classic to Colocation](https://github.com/ember-codemods/ember-component-template-colocation-migrator)
- [Convert pods to Colocation](https://github.com/ijlee2/ember-codemod-pod-to-octane)
- [Convert to `<template>`](https://github.com/IgnaceMaes/ember-codemod-template-tag)


Specifically, these layouts are no longer supported

### Classic 

```
{app,addon}/
  components/
    foo.js
    namespace/
      bar.js
  templates/
    components/
      foo.hbs
      namespace/
        bar.hbs
```

### Pods' Component layout.

```
{app,addon}/
  components/
    foo/
      component.js
      template.hbs
    namespace/
      bar/
        component.js
        template.hbs
```

The above example(s) could fairly easily be migrated (by users) to:

```
{app,addon}/
  components/
    foo.js 
    foo.hbs
    namespace/
      bar.js
      bar.hbs
```

Or using `component-structure=nested`

```
{app,addon}/
  components/
    foo/
      index.js 
      index.hbs
    namespace/
      bar/
        index.js
        index.hbs
```

### `ember-source`

The blueprint for components will need to remove the `classic` option:

Here is the output at the time of authoring this RFC:
```bash 
❯ pnpm ember g component --help
Requested ember-cli commands:

ember generate <blueprint> <options...>
  Generates new code from blueprints.
  aliases: g
  --dry-run (Boolean) (Default: false)
    aliases: -d
  --verbose (Boolean) (Default: false)
    aliases: -v
  --pod (Boolean) (Default: false)
    aliases: -p, -pods
  --classic (Boolean) (Default: false)
    aliases: -c
  --dummy (Boolean) (Default: false)
    aliases: -dum, -id
  --in-repo-addon (String) (Default: null)
    aliases: --in-repo <value>, -ir <value>
  --lint-fix (Boolean) (Default: true)
  --in (String) (Default: null) Runs a blueprint against an in repo addon. A path is expected, relative to the root of the project.
  --typescript (Boolean) Generates a version of the blueprint written in TypeScript (if available).
    aliases: -ts

      component <name> <options...>
        Generates a component.
        --path (String) (Default: components)
          aliases: --no-path (--path="")
        --component-class (@ember/component, @glimmer/component, @ember/component/template-only, "") (Default: --no-component-class)
          aliases: -cc (--component-class=@ember/component), -gc (--component-class=@glimmer/component), -tc (--component-class=@ember/component/template-only), -nc (--component-class=""), --no-component-class (--component-class=""), --with-component-class (--component-class=@glimmer/component)
        --component-structure (flat, nested, classic) (Default: flat)
          aliases: -fs (--component-structure=flat), -ns (--component-structure=nested), -cs (--component-structure=classic)

```

Specifically, we'll remove:
- `--component-structure=classic` and its alias


## How We Teach This

Guides and docs already don't mention the above old layouts.

## Drawbacks

If upgrading to ember v6 without any changes,
- Some super old addons may break
- Some super old apps may break    

In either case, it's been ~4 years since co-located components were introduced, and the deprecation will give folks actionable information for how to move forward (ideally with a link to the deprecation site which can then link to tools folks can try).

## Alternatives

none

## Unresolved questions

none
