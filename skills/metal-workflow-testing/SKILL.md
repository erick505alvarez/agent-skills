---
name: metal-workflow-testing
description: Applies when a user asks to regression-test a Metal workflow, generate a test matrix, validate a workflow change systematically, or audit test coverage for a workflow. Uses universal patterns and past-run gap analysis to build a comprehensive test plan.
version: 3.0.8
---

# Metal Workflow Testing

Generate a systematic test matrix for a Metal workflow, execute dry runs for each scenario, and report per-scenario pass/fail with customer-quality evaluation.

This skill owns test matrix generation and result evaluation. For dry-run execution rules, follow the `metal-workflow-running` skill.

**REQUIRED EVALUATOR:** Every created dry run must receive one fresh-context evaluation from the `metal-workflow-persona-reviewer` agent. Dispatch independent run reviews in parallel. An input-validation rejection that creates no run is the only exception. Self-evaluation does not satisfy this requirement. If the reviewer agent is unavailable or cannot inspect the run, mark that scenario `BLOCKED`; never substitute the parent agent's quality scores.

**HARD-GATE:** Do not start a dry run until you have completed Process steps 1-4 (read definition, scan past runs, generate matrix, gap analysis) and presented the matrix to the user. Skipping the matrix and running a single ad-hoc dry run is the failure mode this skill exists to prevent.

## Connection

- This skill operates through a connected Metal MCP. If more than one Metal MCP is connected, pick one for the task and do not mix them.
- Servers differ by tool surface, not workflow tool behavior: a server that exposes org-selection tools (`get_orgs`, `select_org`, `get_current_org`) serves multiple orgs; a server without them is bound to a single org.
- On a single-org server: if tools are missing or fail to connect, the Metal MCP may not be connected, may be disabled for the org, or the user may be authed against a different org. Ask the user to confirm before concluding anything is broken, and direct them to re-auth when access looks wrong. If tools work but a workflow cannot be found or run, a not-found response can be a policy denial: the workflow exists but is out of the current org's access scope. Ask the user to confirm org access before concluding the workflow does not exist.
- On a multi-org server: every workflow tool fails with "missing org context" until an org is bound with `select_org` (find it with `get_orgs`, verify with `get_current_org`). The binding can expire mid-conversation; the same error later means re-select, not a broken connection. Access-policy gates are broadly bypassed on this server, so a not-found response usually means the wrong org is bound or the resource genuinely does not exist.
- Never proceed against an org the thread has not confirmed; this includes the org passed to `select_org`.

## Process

### 1. Read the Definition

Use `get_workflow` to fetch the workflow's `inputSchema`, step types, branch conditions, HITL steps, iterator sources, and provider connections. This tells you what the workflow can do.

### 2. Scan Past Runs

Use `list_workflow_runs` with `resourceType: "workflow"` and `resourceId` set to the workflow ID. Filter to `COMPLETED` and `FAILED` statuses. For each run, extract:
- Input values used.
- Path taken (which branch arms, which steps completed/failed).
- Terminal status.
- `workflowVersion` (to detect definition drift).

### 3. Generate the Matrix

Load `references/test-patterns.md` and apply scenario templates based on the definition:

- **Input coverage**: each required field with a valid value; each optional field present and absent.
- **HITL paths**: approve, reject, field-edit.
- **Branch paths**: both `then` and `else` arms.
- **Iterator edges**: 0 items, 1 item, N items.
- **Template robustness**: inputs that might cause `<no value>` renders.
- **Provider scenarios**: load the domain pack from `references/domains/<provider>-testing.md` when the workflow targets that provider.

### 4. Gap Analysis

Compare the generated matrix against past run history:

- **Covered** -- a past run exercised this scenario and the outcome was correct.
- **Covered but suspect** -- a past run exercised this scenario but had bugs or quality issues.
- **Uncovered** -- no past run exercised this scenario.

If no past runs exist, generate the full matrix and report: "No past runs found. Generating full matrix from definition patterns."

If past runs use a different `workflowVersion` than the current definition, flag the drift: "Past runs used definition version N, current is version M. Coverage assessment may be stale."

### 5. Prioritize and Execute

Present the matrix. Then execute all uncovered and suspect scenarios without asking which to run. Err on the side of running more scenarios, not fewer. The user asked for thorough testing; asking them to pick scenarios defeats the purpose.

### 6. Execute

Execute every scenario. You may dispatch execution subagents for parallelism, but you must wait for each one to complete and verify it created runs. A subagent that returns without creating runs has not executed; retry it or take over that scenario yourself. Never count an announced plan as execution.

For each selected scenario, follow the `metal-workflow-running` skill's rules for dry-run execution:

1. Resolve the inputs per the scenario specification.
2. Start a dry run with `dryRun: true`.
3. Monitor through completion, resolving HITL steps per the scenario's expected outcome. Poll with the server-requested interval. Do not return control between scenarios; move to the next scenario only after the current one reaches a terminal status.
4. Record the run ID, terminal status, and any errors.
5. Do not stop at the first failure -- run every selected scenario.

If a dry run fails to start, repair against `effectiveInputSchema` and retry once. If it still fails, mark the scenario as "blocked" and continue.

If a dry run hangs, apply the running skill's polling rules. On timeout, mark the scenario as "timeout" and continue.

### 7. Evaluate With Independent Reviewers

After all created runs are terminal, dispatch exactly one `metal-workflow-persona-reviewer` agent per run. Reviews are independent, so dispatch them in parallel. Give each reviewer only:

- org ID;
- workflow ID and version;
- run ID;
- scenario name;
- expected observable outcome from the matrix.

Do not give the reviewer the developer's conclusions or proposed verdict. The reviewer independently inspects Metal MCP evidence and scores:

1. Data Grounding -- citations, tool-call traceability, no hallucination.
2. Output Readability -- human-readable names, formatted figures, no raw IDs.
3. Completeness -- all configured output sections have content.
4. Correctness -- data matches input, internally consistent, no cross-contamination.

A reviewer failure gets one retry. If it still cannot inspect the run or return all four dimensions, mark the scenario `BLOCKED`. The parent agent may verify scenario-specific control flow, but it must not invent, replace, or override the independent quality scores.

A scenario has no final verdict until its reviewer result exists. Input-validation rejections that create no run are reported from the rejection evidence and use Reviewer = `N/A (no run created)`.

### 8. Report

Per-scenario report:

| Scenario | Run ID | Status | Reviewer | Grounding | Readability | Completeness | Correctness | Verdict |
|----------|--------|--------|----------|-----------|-------------|--------------|-------------|---------|
| ... | ... | ... | `metal-workflow-persona-reviewer` or N/A | Pass/Fail/Warn/N/A | Pass/Fail/Warn/N/A | Pass/Fail/Warn/N/A | Pass/Fail/Warn/N/A | Pass/Fail/Blocked |

Highlight failing scenarios with the specific evidence that failed. Cite the reviewer result for every created run. Do not report the test campaign complete while any created run lacks its reviewer result.

## References

- `references/test-patterns.md`: universal scenario templates.
- `references/domains/dealcloud-testing.md`: DealCloud-specific scenarios (load only for DealCloud workflows).
- `references/domains/salesforce-testing.md`: Salesforce-specific scenarios (load only for Salesforce workflows).
- `references/domains/affinity-testing.md`: Affinity-specific scenarios (load only for Affinity workflows).

## Boundaries

- This skill never edits workflow definitions. Route edits to `metal-workflow-building`.
- This skill never starts live (non-dry) runs. All test runs are dry runs.
- This skill does not diagnose failures in depth. Route deep diagnosis to `metal-workflow-debugging`.
