---
name: honeybase-graphql-api
description: Use the Honeybase GraphQL API with an organization API key (hbk_...) — tasks, processes/workflows, runs, teams, schedules, availability, agents. Honeybase is a workflow-automation platform with no public documentation; AI models are not trained on it, so ALWAYS load this skill before writing any query, mutation or integration against a Honeybase server. Includes the full GraphQL schema as a greppable reference.
version: 2026-09-06
---

# Honeybase GraphQL API

Honeybase is a workflow-automation platform (processes → released workflows → runs, plus human tasks, teams, schedules/availability, and AI agents). This skill is the API documentation — nothing about this API is public or in your training data. Do not guess field or operation names: verify against the bundled schema.

## Endpoint & auth

```
POST {BASE_URL}/api/graphql
Authorization: Bearer hbk_<key>
Content-Type: application/json
Body: {"query": "...", "variables": {...}}
```

- `BASE_URL` is the org's server (e.g. `https://app.honeybase.ai`, or `http://localhost:8080` in dev). Ask the user if unknown.
- The key is an **organization API key** created in *Organization Settings → Developer → API Keys*. It is org-scoped and carries an explicit permission set chosen at creation. It is NOT a JWT and never expires mid-session — no refresh dance, just send it on every request.
- A wrong/expired/revoked key → HTTP **401** (uniform; the server won't tell you which). A missing permission → HTTP 200 with an error entry (below). Keys can NEVER manage API keys, rename the org, or do anything the schema marks "Requires Owner or Admin role".

## Token discipline (read this before querying)

1. **Grep `references/schema.gql` instead of introspecting.** The full SDL ships with this skill. `grep -n -A15 '^type TaskMutations'` costs ~20 lines; a full remote introspection costs tens of thousands of tokens. Only fall back to introspection if the target server is likely newer than this snapshot, and then scope it: `{"query":"{ __type(name: \"TaskQueries\") { fields { name } } }"}` — names only, one type at a time, never `__schema` wholesale.
2. **Select only the fields you need.** Never copy a whole type's field list into a selection "just in case".
3. **Ask for small pages.** List operations take `limit`/`offset` (server clamps anyway). Start with `limit: 10`.
4. **Truncate responses before they enter your context**: `curl ... | head -c 2000`, or `| jq '.data.tasks.listTasks | length'` when you only need a count.
5. **Prefer point lookups** (`getTask(taskId:)`, `getProcess(publicId:)`) over list-and-filter.

## Shape of the API

Everything is namespaced one level under the roots. Top-level `Queries` / `Mutations` fields (each resolves to a namespace object whose fields are the operations):

| Namespace | What's in it |
|---|---|
| `tasks` | human work items: list/get/create/update/delete, time series, audit trail (`taskUpdates`) |
| `processes` | processes, workflows (draft/released), runs (`listProcessRuns`, `getProcessRun`), stats |
| `teams` | teams + membership |
| `users` | org members, roles |
| `schedules`, `availability` | job schedules, availability state/history, reassignment |
| `agents`, `tools`, `aiChat` | AI agents, their runs and tools |
| `integrations`, `secrets`, `orgDatabase`, `forms`, `taskViews`, `notifications`, `organize` | as named |
| `apiKeys`, `organizations`, `oauth`, `auth` | admin/session surfaces — **not usable with an API key** |

Conventions:
- **Ids are `Long`**; most top entities also have an opaque `publicId` (UUID string). By-id getters dual-accept: pass exactly ONE of `taskId:` / `publicId:`.
- **Dates**: `LocalDate` = `"2026-07-31"`, `ZonedDateTime` = ISO-8601 (`"2026-07-31T12:00:00Z"`). Entity timestamps come back as ISO-8601 strings.
- **`JsonString` fields** (e.g. task `metadata`, integration configs) are JSON **encoded as a string** — double-escape when sending, parse after reading.
- **Permissions**: every operation's docstring in the SDL ends with what it requires (`Requires canManageTasks.`, `Requires Owner or Admin role.`). The key only has the flags granted at creation; grep the docstring before calling and don't retry a `PERMISSION_DENIED`.

## Errors

GraphQL errors arrive as HTTP 200 with the failed field `null` plus:

```json
{"errors":[{"path":["tasks","createTask"],
  "extensions":{"errorType":"PERMISSION_DENIED","errorMessage":"Permission required: canManageTasks"}}]}
```

`errorType` ∈ `AUTHENTICATION_ERROR` (no/insufficient auth) · `PERMISSION_DENIED` (key lacks the flag — don't retry) · `CONDITION_ERROR` (bad input/state — fix the input) · `EFFECT_ERROR` (server-side failure) · `VALIDATION_ERROR`/`PARSING_ERROR` (your query text is wrong — re-check against the schema). Read `errorMessage`: it is written to be user-facing.

## Worked examples

List open tasks:
```bash
curl -s $BASE/api/graphql -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" -d '{"query":
  "query { tasks { listTasks(includeCompleted: false) { taskId name priority dueDate assignedTo } } }"}'
```

Create a task (requires `canManageTasks`; assign to a user OR let a team pick):
```bash
-d '{"query":"mutation($n: String!, $d: LocalDate!, $u: Long) { tasks { createTaskWithAssignment(name: $n, dueDate: $d, priority: \"Medium\", userId: $u) { taskId publicId } } }",
     "variables": {"n": "Follow up", "d": "2026-08-01", "u": 12}}'
```

Run history of a process, newest first:
```bash
-d '{"query":"query { processes { listProcessRuns(processId: 5, limit: 10, offset: 0) { runWorkflowId phase startedAt } } }"}'
```

Who can be assigned (org members):
```bash
-d '{"query":"query { users { listMembers { webUserId firstName lastName email } } }"}'
```

Every task write made with the key is attributed to the key by name in the task's audit trail (`tasks { taskUpdates(taskId:) }`) — you don't need to (and can't) impersonate a user.

## Reference

- `references/schema.gql` — the full SDL. Discovery recipe: `grep -n '^type Queries\|^type Mutations' references/schema.gql` for the roots → `grep -n -A30 '^type <Namespace>Queries'` for operations → `grep -n -B2 -A20 '^type <Entity> '` for fields. Docstrings (in `"..."` above fields) carry argument semantics and required permissions.
- **Human-readable docs** at `https://docs.honeybase.ai` complement this bundle: the GraphQL guides at `https://docs.honeybase.ai/api/` (authentication, conventions, pagination, errors, plus a generated schema reference) and integration setup at `https://docs.honeybase.ai/integrations/` (what each connection's form asks for). Use them for prose and context; keep grepping `references/schema.gql` for authoritative field/operation names.

## Sharing this skill

The folder `.claude/skills/honeybase-graphql-api/` is self-contained — copy it into any project's `.claude/skills/` (or a user's `~/.claude/skills/`) and Claude will load it when asked to work with the Honeybase API. It is also hosted at `https://honeybase.ai/skills/honeybase-graphql-api/` (SKILL.md, references/schema.gql, version.json — compare `version.json` with the `version:` line above to know when to re-download); the MCP `get_graphql_api_skill` tool hands out those URLs. Pair it with an API key scoped to only the permissions the integration needs.
