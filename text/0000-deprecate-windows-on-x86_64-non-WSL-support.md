---
stage: accepted
start-date: # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
suite: 
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
suite: Leave as is
-->

<-- Replace "RFC title" with the title of your RFC -->
# Drop support for Windows 

## Summary

Testing Windows in CI on GitHux is 3-7x slower than testing on Mac or Ubuntu. 
Additionally, Windows' file system is quirky, and a lot of time goes in to keeping compatibility because _we support Windows_ right now.

This RFC proposes dropping the need to test Windows on CI for the ember family of projects. 
(and indirectly dropping support for Windows without WSL or a "Dev Drive")
> One paragraph explanation of the feature.

## Motivation

Testing Windows in CI on GitHux is 3-7x slower than testing on Mac or Ubuntu. 
On average, this adds up to a 3.5 to 4x increase in CI time on the ember-cli repo, resulting in much slower time to land PRs, since the repo requires PRs be upt to date (or merge queue)[^ci-health]

There is [anecdotal evidance](https://x.com/wojtekmaj91/status/1810741430413054300) that Windows with an ARM architecture does not have such performance issues in JavaScript projects, but GitHub [is not planning to add ARM x64 Support](https://github.com/actions/runner-images/issues/768)

[^ci-health]: without either PRs be up to date (or a merge queue), you can have race conditions in your PRs that accidentally break `main`. Unless a repo has its PRs always rebased, the requirements of ember-cli's repo specifically are good practice for any repo with multiple contributors. Example of where not having this will break `main`: one person merges a lint change, but another active PR hasn't rebased and currently would violate that lint change -- but CI is green and is merged anyway -- failure is not realized until `main`s CI runs. 


When submitting many small PRs to a project, such as ember-cli, the slowness of CI results in ultimately what amounts to a collective menial change (dependency updates) take all day to watch, manage, rebase, retry merge, etc. 

Over the course of 8 PRs, CI takes 46+ minutes for each one, and an additional 46+ minutes for each PR that will be merged "next" -- if Windows testing were removed (with no other changes to CI), we'd save a total of


(all times in minutes, sourced f rom [this CI](https://github.com/ember-cli/ember-cli/actions/runs/9862588536))

```
# (1) => (10.5 + 32)
#        (longest times for each stage of CI) 
#        42.5
# (2) => (1) * 8 
#        340      # initial amount of time all PRs take in CI
# (3) => (1) * 7 
#        297.5      # minimal number of re-runs, for manual updating of a PR 
# (4) => (3) + (2)
#        637.5      # minimal amount of CI time, if no flakey tests run
```


And without windows, this would be: 
```
# (1) => (1.9 + 12)
#        13.9
# (2) => (1) * 8 
#        111.2      # initial amount of time all PRs take in CI
# (3) => (1) * 7 
#        97.3      # minimal number of re-runs, for manual updating of a PR 
# (4) => (3) + (2)
#        208.5      # minimal amount of CI time, if no flakey tests run
```

Total improvement: 3.06xx (without windows takes 33% of the time)



> Why are we doing this? What use cases does it support? What is the expected
outcome?

## Detailed design

This is a WIP RFC, and I'm more curious about what the community thinks if this is a bad idea, and if official windows support should be kept or not.

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here. 

> Please keep in mind any implications within the Ember ecosystem, such as:
> - Lint rules (ember-template-lint, eslint-plugin-ember) that should be added, modified or removed
> - Features that are replaced or made obsolete by this feature and should eventually be deprecated
> - Ember Inspector and debuggability
> - Server-side Rendering
> - Ember Engines
> - The Addon Ecosystem
> - IDE Support
> - Blueprints that should be added or modified

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

> Keep in mind the variety of learning materials: API docs, guides, blog posts, tutorials, etc.

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
