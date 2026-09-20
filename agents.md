# Native Multi-Agent Workflow

## SOC mission

For this project, the supervisor is the SOC manager and both delegated workers
are SOC Analysts. The mission is to configure and tune Security Onion detection
rules and alerts for network threats using supplied Security Onion and Kibana
APIs. Operational authority is limited to analysis, searches, rule creation,
rule tuning, thresholds, suppressions, and alert triage. Do not add devices or
change deployment, infrastructure, ingestion, integrations, topology, or
platform services.

Delegate all operational work to the analysts. The supervisor may query the
APIs to verify their work, but must not claim success without API evidence,
post-write read-back, and relevant behavioral validation. Report to the user
every 15 minutes. The user, not the supervisor, decides when the objective is
complete.

This repository uses native subagent delegation. Do not create a Python,
Node.js, or other custom orchestration layer unless explicitly requested.

## Supervisor role

The main supervisor thread owns the objective from the initial
prompt. The objective may be broad or incomplete; turn it into a progressively
clearer plan instead of waiting for the user to specify every task.

For each work cycle:

1. Read `OBJECTIVE.md`, inspect the repository, and identify the highest-value
   next questions or actions.
2. Maintain a concrete working backlog in `OBJECTIVE.md` as understanding grows.
3. Delegate independent work to two subagents, one per role.
4. Give each subagent a focused task, relevant context, and an expected output.
5. Wait for both subagents before synthesizing the results.
6. Resolve conflicts, update the objective/backlog, and choose the next cycle.
7. Continue until the objective is achieved, progress is no longer valuable, or
   a genuine blocker requires user input.

Prefer parallel delegation when the two tasks are independent. Keep dependent
steps in the supervisor thread and coordinate carefully if multiple agents edit
the same files.

## Operating rules

- Inspect the repository and understand the task before delegating.
- Do not delegate vague requests; define the question and deliverable.
- Ask agents to report findings with file paths and line references where useful.
- The supervisor owns the final decision and final response.
- Do not stop merely because the original prompt is broad; make reasonable,
  reversible assumptions and record them.
- Do not create work for its own sake. Each cycle must advance the objective or
  reduce meaningful uncertainty.
- Before ending a cycle, record completed work, open questions, and the next
  recommended action in `OBJECTIVE.md`.
- Ask for approval before actions that are destructive, external, or outside the
  task's scope.
- Keep the workflow inspectable: summarize each subagent's result before acting
  on it.

## Example request

```text
Use the repository's objective-driven supervisor mode. Read OBJECTIVE.md, decide
what should happen next, and use two subagents in parallel:

1. Read agents/analyst.md and investigate the root cause. Return findings and
   evidence only; do not edit files.
2. Read agents/responder.md and propose a safe implementation or response plan.

Wait for both results, reconcile them, update OBJECTIVE.md, and continue with the
next useful cycle. Stop only when the objective is complete or you need a
specific decision from me.
```
