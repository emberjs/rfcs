---
stage: accepted
start-date: 2026-01-26T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams:
  - learning
  - cli
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

Agent skills are supplemental to the official Guides. They should never be treated as a replacement for canonical documentation, and they are maintained for the current Ember version only (not as a historical, multi-version compatibility reference).

Also Agent skills can cover more topics than the official guides, have more opinions, etc.

## Motivation

AI assistants are already part of day-to-day development. The ecosystem needs a way to:

- Share Ember-specific guidance in a format agents can reliably consume
- Keep that guidance updated through normal OSS workflows (PR review, changelog, release cadence)
- Make it easy for teams to standardize how agents review Ember code (performance, accessibility, conventions)

The `agent-skills` project provides an existing, interoperable format (agentskills.io) for packaging agent instructions and helper scripts.

The fork at https://github.com/NullVoxPopuli/agent-skills already includes Ember-focused content (notably an “ember-best-practices” skill) and supporting build/validation tooling.
### Skills benefit humans too

While the packaging format is designed for agent consumption, the actual content—best practices, patterns, architectural guidance—is equally valuable to human developers. Skills partially supersede the [Cookbook RFC](https://github.com/emberjs/rfcs/issues/786) and could eventually be rendered on a site like [cookbook.emberjs.com](https://cookbook.emberjs.com/) so the same knowledge base serves both audiences.

One might ask: why not just write documentation for humans and let LLMs discover it? Skills complement official documentation rather than replace it. The official Guides and API docs remain canonical. Skills add value by:

- Providing opinionated, action-oriented instructions tuned to agent workflows (e.g. "review this component for accessibility regressions")
- Bundling implicit knowledge that experienced Ember developers carry but that isn't always captured in reference docs
- Pointing agents to the right canonical docs at the right time, rather than leaving them to search the entire surface area

Where gaps exist in official documentation, skills can serve as a forcing function to identify and fix them upstream.

If one day agent-skills are no longer needed by agents, these docs are still useful for humans, training, reference, ideas, etc.

### Why adopt now?

Skills may not be the dominant AI integration pattern forever. However, there is demonstrated value today: community testing has shown that injecting Ember skills into a local model (e.g. qwen3-coder-next) improved output from roughly 70% correct (and confusing to newcomers) to nearly fault-free. Moving quickly lets the Ember ecosystem benefit now while the format is actively evolving (which [it is](https://agentskills.io/home), and the format generated into a project changes over time to use better techniques for ai token efficiency).

We are prioritizing moving quickly and will not be guaranteeing any format stability within the agent-skills repo, but will put forth a best-effort for compatibility with the skill-generating/consuming tools available. 

This is not promising the same stability that ember promises.

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

**Scope clarification:** This RFC is about adopting the skills repo as shared infrastructure. The actual skill contents are a living set of micro-documents and are intentionally out of scope—they will evolve through normal PR workflows without requiring further RFCs. The contribution bar should be low: since AI output quality is inherently fuzzy, any potential improvement is worth trying, and we should not gate contributions on heavy process.

### Scope (initial)

Start with the Ember-focused skill(s) already present in the fork, especially “ember-best-practices” (performance + accessibility oriented), and iterate from there.

Non-Ember skills that happen to live in the repo today can remain if they are useful for web development in general, but the adoption goal is to ensure Ember-centric guidance is first-class.

### Versioning strategy

Ember-focused skills target the **current Ember release** by default. The repo will use LTS-based folders so users can select the skill set appropriate for their project:

```
❯ pnpm dlx skills add ember-tooling/agent-skills

┌   skills
│
◇  Found 2 skills
│
◆  Select skills to install (space to toggle)
│  ◻ ember-release
│  ◻ ember-lts-6.12
└
```

When a new LTS is declared, the current `ember-release` files are copied into a new LTS folder (e.g. `ember-lts-6.12`). This gives LTS users a stable snapshot while `ember-release` continues to evolve.

Backporting skills to much older versions (e.g. Ember 4) is possible if there is community desire, but it would require reorganizing skills by feature (e.g. template-tag, classic HBS) rather than by version, so that feature-specific guidance isn't incorrectly recommended to users whose projects don't use that feature.

### External content policy

Skills should avoid fetching content from external URLs at runtime. If a skill references third-party guidance (for example, Vercel's Web Interface Guidelines), that content should be **vendored** into the repo so that:

- Skills work offline and in air-gapped environments
- There is no risk of upstream content changing unexpectedly
- All content is auditable through the normal PR review process

Vendored content can be periodically updated via automated PRs that diff upstream changes.

### Token efficiency

Skill authors should follow emerging [agentskills best practices](https://github.com/mgechev/skills-best-practices) (and on [agentskills.io](https://agentskills.io/home)) for token efficiency. Since skills are injected into agent context windows, every token matters. Guidance should be concise, structured, and free of boilerplate.

### Installation / usage

Agent Skills are installed from a repository reference. The recommended command once the repo is adopted:

```sh
pnpm dlx skills add ember-tooling/agent-skills
```

(This uses the [skills CLI](https://github.com/vercel-labs/skills?tab=readme-ov-file#skills).) Once installed, compatible agents can auto-apply skills when prompted.

We should document the “happy path” using the `ember-tooling` repo location once adopted.

## How we teach this

- Add a short Guides section: “AI-assisted development: Skills” explaining what skills are, when to use them, and how to install them.
- Provide 2–3 concrete examples:
  - “Review this Ember route for data fetching issues”
  - “Audit this template for accessibility regressions”
  - “Suggest improvements for bundle size / build output”
- Cross-link from the `ember-mcp` recommendation: MCP provides data/tools; skills provide opinionated workflows.
- Emphasize that skills are supplemental to the official Guides and API docs—not a replacement. Where skill content reveals gaps in canonical docs, those gaps should be fixed upstream.
- Consider rendering skill content on [cookbook.emberjs.com](https://cookbook.emberjs.com/) so the same knowledge base is browsable by humans in a web-friendly format.

## Drawbacks

- Maintenance burden: best practices evolve, and a “recommended” skill set must keep pace.
- Risk of over-prescription: skills can encode opinions; we’ll need clear language about what’s “recommended” vs “optional.”
- Safety: skills may include scripts; maintainers should be conservative about automation and emphasize review of actions.
- Longevity: the "skills" format may be superseded as AI tooling matures. However, the underlying knowledge is valuable regardless of packaging, and the low-overhead nature of skills means the investment is proportionate to the risk.
- Overlap with existing docs: skill content may duplicate guidance already in the Guides or API docs. This is acceptable as long as skills point back to canonical sources and doc gaps discovered through skill authoring are fed back upstream.

## Alternatives

- Keep everything ad-hoc in blog posts / gists (hard to discover, not versioned, not enforceable).
- Rely solely on generic AI models plus docs (agents still miss context, and there’s no shared review surface).
- Build Ember-specific integrations per editor/provider (more work, less portable).

## Unresolved questions

- Governance: which group owns reviews and releases (Tooling team, Learning team, or shared working group)?
- Release model: tags only, GitHub releases, or also published packages? (LTS-based folder snapshots are planned, but the exact release mechanics need definition.)
- Policy: what skills qualify as "recommended" (and how do we deprecate or supersede skills)?
- Security posture: what constraints do we place on including scripts (network access, destructive commands, sandboxing guidance)? All external content should be vendored rather than fetched at runtime.
- Backporting: how far back do we support? The default is current-release-only, but community demand for older LTS or Ember 4 support would require feature-based skill organization.
- Cookbook RFC integration: should skill content also be rendered on [cookbook.emberjs.com](https://cookbook.emberjs.com/), and if so, what is the build/publishing pipeline?
