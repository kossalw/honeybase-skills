# Honeybase agent skills

Installable [agent skills](https://docs.claude.com/en/docs/claude-code/skills) that teach
an AI coding agent (Claude Code, Cursor, Codex, and other tools that read
`.claude/skills/`) how to work with [Honeybase](https://honeybase.ai) — a workflow-automation
platform whose API, MCP server, and form schema aren't in any model's training data.

## Install

Using the open-source [`skills`](https://github.com/vercel-labs/skills) CLI:

```bash
# All three skills
npx skills add kossalw/honeybase-skills

# Just one
npx skills add kossalw/honeybase-skills --skill honeybase-graphql-api

# Install for every project
npx skills add kossalw/honeybase-skills --global
```

Or copy any `skills/<name>/` folder into your project's `.claude/skills/` by hand.

## The skills

| Skill | Teaches an agent to… |
| --- | --- |
| [`honeybase-mcp`](skills/honeybase-mcp) | Drive Honeybase over its remote MCP server (OAuth, scopes, tool discovery). |
| [`honeybase-graphql-api`](skills/honeybase-graphql-api) | Use the Honeybase GraphQL API with an organization API key. Bundles the full schema. |
| [`honeybase-forms`](skills/honeybase-forms) | Author the subsetted SurveyJS form definitions Honeybase accepts. |

## Docs

Human-readable documentation lives at **[docs.honeybase.ai](https://docs.honeybase.ai)**
(concepts, the [GraphQL API](https://docs.honeybase.ai/api/), the
[MCP server](https://docs.honeybase.ai/mcp/), and [these skills](https://docs.honeybase.ai/skills/)).

---

*This repository is generated — the skills are mirrored here automatically. File issues
and feature requests through Honeybase support.*
