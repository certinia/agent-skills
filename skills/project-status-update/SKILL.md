---
name: project-status-update
description: Produce a status update for a single Certinia PSA project (pse__Proj__c) by running a fixed set of checks against the "Certinia" Salesforce connector. Use this whenever the user asks for an update, status, health check, or "how is it going" on a specific project, or asks about its at-risk milestones, missing timecards, tasks, assignment hours variance, or risks, even if they don't name those checks. For a whole portfolio of projects, use portfolio-risk-review instead.
---

# Project Status Update

When a user asks for an update on a project, run five checks against the **Certinia** Salesforce connector and combine the results into one status summary. Each check surfaces a different kind of problem (schedule, time entry, task progress, staffing, logged risks), so skipping one leaves a blind spot in the update.

## Before you start

The connector's tools are deferred, so load them first with `tool_search` (for example the queries "pse MilestoneQueryAction", "pse TimecardQueryAction", "pse ProjectTaskAnalysisAction", "pse AssignmentHoursVarianceAction", and "soqlQuery"). Don't guess parameter names; use the schemas that come back.

If the user named a project, find its record Id first (a `find` or SOQL query on `pse__Proj__c` by name). If they didn't name one, ask which project. If the name matches more than one project, list the candidates and ask.

## The five checks

Run these for the project. They are independent, so run them together where you can.

1. **Milestones at risk**: `pse_MilestoneQueryAction`. Identifies milestones that are late, due soon, or otherwise at risk.
2. **Missing timecards**: `pse_TimecardQueryAction`. Identifies resources on the project who haven't submitted timecards.
3. **Project tasks**: `pse_ProjectTaskAnalysisAction`. Identifies the project's tasks and their status (overdue, not started, in progress, complete).
4. **Assignment hours variance**: `pse_AssignmentHoursVarianceAction`. Identifies assignments whose actual hours diverge from planned hours, high or low.
5. **Risks**: query `pse__Risk__c` records related to the project (SOQL via `soqlQuery`, or `getRelatedRecords` from the project). Use `describeSObject` or `getObjectSchema` to confirm field names rather than assuming them. Pull at least the name, status, and any impact or probability fields that exist.

If one check fails or returns nothing, say so in the summary rather than dropping it silently. "No risks logged" and "the risk query failed" mean very different things to the reader.

## Output format

Lead with the answer, then the detail. Keep it short enough to read in under a minute.

```
# [Project name] status update

**Overall:** one or two sentences on the project's health and the single most important thing needing attention.

## Milestones at risk
## Missing timecards
## Project tasks
## Assignment hours variance
## Risks
## Suggested next steps
```

- Under each heading, list only items that matter. If a check came back clean, write one line saying so.
- Name the specific milestones, people, tasks, and assignments, with dates and hours where available, so the reader can act without opening Salesforce.
- Close with two to four concrete next steps drawn from the findings (e.g. chase named people for timecards, replan a milestone).
- Report only what the data shows. Don't speculate about causes that the records don't support.

## Read-only

This skill is for reporting. Don't create, update, or delete any records while producing the update. If the findings suggest a change (like logging a new risk), propose it in the next steps and wait for the user to ask.
