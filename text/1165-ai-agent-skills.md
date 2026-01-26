---
stage: accepted
start-date: 2026-01-26T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams:
  - learning
  - tooling
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1165
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

# Adopt Agent Skills for Ember Tooling

## Summary

Adopt the Agent Skills repository (https://github.com/NullVoxPopuli/agent-skills) under the `ember-tooling` GitHub organization and treat it as the canonical home for Ember-focused “skills” for AI coding agents.

In practice, this gives the community a stable, reviewable way to package Ember best practices and lightweight automation for agents, similar in spirit to how we treat lint rules and guides as shared infrastructure.

## Motivation

AI assistants are already part of day-to-day development. The ecosystem needs a way to:

- Share Ember-specific guidance in a format agents can reliably consume
- Keep that guidance updated through normal OSS workflows (PR review, changelog, release cadence)
- Make it easy for teams to standardize how agents review Ember code (performance, accessibility, conventions)

The `agent-skills` project provides an existing, interoperable format (agentskills.io) for packaging agent instructions and helper scripts.

The fork at https://github.com/NullVoxPopuli/agent-skills already includes Ember-focused content (notably an “ember-best-practices” skill) and supporting build/validation tooling.

## Detailed design

### What is an “Agent Skill”?

An Agent Skill is a small, versioned bundle of:

- Agent-facing instructions (for example, how to review code or how to run a task)
- Optional helper scripts for automation
- Optional references

The intention is that an agent can automatically apply a skill when it detects the task is relevant (e.g. “Review this Ember component for performance issues”).

### Proposal

1. Transfer or mirror the repository to `ember-tooling/agent-skills`.
2. Treat Ember-related skills in that repo as recommended defaults for Ember workflows.
3. Establish a lightweight review and release process so changes to skills are:
   - auditable (PR review)
   - testable (CI/validation)
   - versioned (tags/releases)

This RFC does not require any one editor or AI provider. The point is to maintain shared content in a vendor-neutral format.

### Scope (initial)

Start with the Ember-focused skill(s) already present in the fork, especially “ember-best-practices” (performance + accessibility oriented), and iterate from there.

Non-Ember skills that happen to live in the repo today can remain if they are useful for web development in general, but the adoption goal is to ensure Ember-centric guidance is first-class.

### Installation / usage

Agent Skills are installed from a repository reference (for example via tooling that supports []`npx add-skill …`](https://github.com/vercel-labs/skills?tab=readme-ov-file#skills)). Once installed, compatible agents can auto-apply skills when prompted.

We should document the “happy path” using the `ember-tooling` repo location once adopted.

## How we teach this

- Add a short Guides section: “AI-assisted development: Skills” explaining what skills are, when to use them, and how to install them.
- Provide 2–3 concrete examples:
  - “Review this Ember route for data fetching issues”
  - “Audit this template for accessibility regressions”
  - “Suggest improvements for bundle size / build output”
- Cross-link from the `ember-mcp` recommendation: MCP provides data/tools; skills provide opinionated workflows.

## Drawbacks

- Maintenance burden: best practices evolve, and a “recommended” skill set must keep pace.
- Risk of over-prescription: skills can encode opinions; we’ll need clear language about what’s “recommended” vs “optional.”
- Safety: skills may include scripts; maintainers should be conservative about automation and emphasize review of actions.

## Alternatives

- Keep everything ad-hoc in blog posts / gists (hard to discover, not versioned, not enforceable).
- Rely solely on generic AI models plus docs (agents still miss context, and there’s no shared review surface).
- Build Ember-specific integrations per editor/provider (more work, less portable).

## Unresolved questions

- Governance: which group owns reviews and releases (Tooling team, Learning team, or shared working group)?
- Release model: tags only, GitHub releases, or also published packages?
- Policy: what skills qualify as “recommended” (and how do we deprecate or supersede skills)?
- Security posture: what constraints do we place on including scripts (network access, destructive commands, sandboxing guidance)?
