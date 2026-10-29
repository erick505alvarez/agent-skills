---
name: linear
description: Use when reading, creating, or updating Linear issues, projects, and comments through MCP, or when expanding a pasted issue link into a context brief.
---

# Linear

## Overview

Manage Linear issues, projects, and comments through the Linear MCP server. Use the MCP tool schema as the source of truth. Read first, then write only the fields you change.

## Setup

If a call fails because the Linear MCP is not connected:

1. `codex mcp add linear --url https://mcp.linear.app/mcp`
2. Enable the remote MCP client: set `[features] rmcp_client = true` in `config.toml`, or run `codex --enable rmcp_client`.
3. `codex mcp login linear`.

Restart after login. On Windows or WSL, if connection errors persist, run the MCP through WSL with `npx -y mcp-remote https://mcp.linear.app/sse --transport sse-only`.

## Select the call surface

| Interface | Call shape |
| --- | --- |
| Generic MCP caller | `{"server":"linear","toolName":"save_issue","arguments":{...}}` |
| Direct generated tool | `tools.mcp__linear__save_issue({...})` |

Do not mix the interfaces. The direct tool parameter object is the `arguments` value. The app connector and direct MCP tools can have separate auth sessions. If the connector reports reauthentication and a direct tool is available, use the direct surface and restart with `get_issue`.

## Ticket context brief

Use when the user shares an issue link or key and wants it summarized or prepared for implementation.

### 1. Normalize the reference

Accept a full issue URL or a short key such as `ENG-123`. Extract the identifier from the URL. If it is unclear, ask for the exact key. A bare URL means the full brief.

### 2. Gather the core issue

- `get_issue` with customer needs and relations included.
- `list_comments` for the issue.

Capture title, description, state, priority, assignee, project, milestone, cycle, labels, due date, branch name, attachments, links, customer needs, duplicate/blocking/blocked-by/related issues, and noteworthy comment details.
- Read an embedded image through `extract_images`; the signed `src` in the
  description or a comment can expire.

### 3. Expand project context

If the issue belongs to a project, call `get_project` with milestones and resources, and `get_status_updates` for recent health. Fetch a linked Linear document with `get_document` when it is central, and summarize only the relevant parts.

### 4. Expand direct relations

Fetch only the neighboring issues that explain the ticket: blockers, blocked-by, duplicates, and explicitly related issues. Do not traverse the whole graph unless the user asks for a dependency audit.

### 5. Collect PR and code context

Treat GitHub, GitLab, or Bitbucket PR links in links, attachments, or comments as code signals, plus the issue branch name. Report each with URL or branch, a short label, and where it was found. If none exist, say so.

### 6. Return a structured brief

Order: issue snapshot, key comments and decisions, project context, project resources and docs, related issues, PRs and branch context, gaps or follow-up questions.

Prefer direct facts. Label inferences. If the user is preparing to work on the issue, add an "Implementation handoff" subsection: likely code area, whether a branch or PR exists, missing information, and the next 1–3 steps.

## Update an issue

1. `get_issue` with the identifier. Record the exact identifier and the fields that must remain unchanged.
2. Inspect the `save_issue` schema in the active tool list. Use its field names exactly.
3. Update with `id` and each field to change. Omit every field that must remain unchanged.
4. Read the issue again. Confirm changed and preserved fields.

If the schema supports `patch` and the target text is a unique exact anchor, use `patch` for a small description edit. Use `description` for a full replacement when no safe anchor exists. Omit `patch` when the schema does not support it.

```json
{
  "server": "linear",
  "toolName": "save_issue",
  "arguments": {
    "id": "ENG-11070",
    "description": "Complete replacement description"
  }
}
```

## Resuming a workflow

Read the live issue before a resumed write. Verify a reported update with `get_issue` and `list_comments` before saving. Do not duplicate a comment that already exists.

## Practical workflows

- Sprint planning: review open issues, pick by priority, create a cycle with assignments.
- Bug triage: list critical bugs, rank by impact, move the top items to In Progress.
- Documentation audit: search docs, open issues for gaps.
- Release planning: create a project with milestones and issues.
- Cross-project dependencies: find blocked issues, identify blockers, create missing links.

## Troubleshooting

- Auth: clear cookies, re-run OAuth, verify workspace permissions.
- Tool errors: confirm required fields and split complex requests.
- Missing data: refresh the token, verify workspace access, check archived projects.
- Rate limits: batch operations and cache frequent queries.

## Common mistakes

| Error | Fix |
| --- | --- |
| `patch: Invalid input` | Omit `patch` for a full replacement. |
| `issue_id`, `issue_key`, or `issue_identifier` | Use `id`. |
| `list_comments({id})` rejected | `list_comments` takes `issueId`; only `get_issue` takes `id`. |
| `get_project({id})` rejected | `get_project` takes `query` (name, ID, identifier, or slug). |
| Properties outside `arguments` | Put all inputs inside `arguments`. |
| Fields change unexpectedly | Read first; send only required changes. |
| Duplicate comment after a handoff | Verify the live issue and comments first. |
| Sending every field to preserve it | Omitted fields remain unchanged. |
