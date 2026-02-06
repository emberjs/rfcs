---
stage: accepted
start-date: 2026-01-26T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams:
  - learning
  - cli
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1164
project-link: 
suite: 
---

# Recommend hosting `ember-mcp` on emberjs.com

## Summary

Recommend `ember-mcp` as the default MCP server for Ember projects. The goal is simple: when people use AI tools, they should get current Ember APIs and modern patterns by default, not legacy advice.

## Motivation

AI assistants are now part of many workflows, but they regularly miss Ember-specific context. The common failures are predictable: classic-era patterns, wrong APIs for the Ember version in use, and shell commands that don’t match the repo’s package manager.

`ember-mcp` addresses this by giving AI tools a small set of focused capabilities:

- Search Ember docs (API, Guides, community)
- Fetch targeted API references
- Provide best-practice guidance for modern Ember
- Provide version/migration info
- Detect the correct package manager and suggest the right commands

The expected outcome is fewer wrong suggestions and less time spent “arguing with the bot.”

## Detailed design

MCP (Model Context Protocol) lets AI tools call external “tools” in a structured way. `ember-mcp` is an MCP server that exposes Ember documentation and related utilities.

### What we’re recommending

- The recommended package is `ember-mcp`.
- The upstream project is https://github.com/ember-tooling/ember-mcp.

### Data source

`ember-mcp` loads and indexes aggregated Ember documentation at startup (API docs, Guides, community content, examples) from:

`https://nullvoxpopuli.github.io/ember-ai-information-aggregator/llms-full.txt`

### Client configuration

Any MCP-compatible client can run it via `npx`:

```json
{
  "mcpServers": {
    "ember": {
      "command": "npx",
      "args": ["-y", "ember-mcp"]
    }
  }
}
```

### “Hosted” expectation

A public endpoint: `https://mcp.emberjs.com` (or similar) for the latest version

And then for versions:
- `https://mcp.emberjs.com/beta`
- `https://mcp.emberjs.com/v6.10.x`
- `https://mcp.emberjs.com/v6.8.x`
- etc


## How we teach this

- Add a short “AI-assisted development” section to the Guides: what MCP is, when to use it, and the one configuration snippet above.
- Add `ember-mcp` to the emberjs.com tooling page alongside Ember Inspector and linting tools.
- Publish a short announcement post focused on practical examples (upgrades, API lookups, best practices).

## Disclaimer and safety

The ember-mcp server is a read-only source of documentation. The server itself does not offer any mutation actions. But like any MCP server, it may suggest a command to your agent that your agent will run, and you need to be responsible for securing your environment against dangerous or destructive actions by your agent.

This includes (but isn’t limited to):

- Deleting code
- Deleting folders/files outside of your project
- Unintended modifications to your codebase
- Other adverse effects (including damage to your OS or machine)

It’s important to be explicit about responsibility: MCP implementations and their maintainers are not responsible for damage caused by running generated commands or applying generated changes.

Practical guidance:

- Review commands and diffs before applying them.
- Use version control and keep backups.
- Prefer running tools with least privileges (and in disposable environments when possible).

## Drawbacks

- Ongoing maintenance: `ember-mcp` needs regular updates as docs and best practices change.
- Trust/safety: it’s a tool surface for AI agents; docs should encourage reviewing output and using version control.
- Dependency on the aggregator URL staying available and current.

## Alternatives

- Do nothing (developers keep getting inconsistent, often outdated Ember advice).
- “Just write better docs” (helps, but doesn’t give AI tools structured access).
- Build editor-specific integrations (more maintenance, less portable than MCP).

## Unresolved questions

- Q: What’s the policy for changes to the doc aggregator source and inclusion criteria?
  - write more docs on emberjs.com
  - write blog posts
