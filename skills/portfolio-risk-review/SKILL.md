---
name: "portfolio-risk-review"
description: Use this skill whenever the user asks about a project portfolio's health or wants to know which projects are at risk, behind schedule, or need attention — e.g. "how's the Acme portfolio looking", "which projects in portfolio X are at risk", "give me a portfolio status review", "any projects behind on milestones". This covers Certinia/PSA project portfolios (the pse__Portfolio__c object) and their related projects (pse__Proj__c). Trigger this even if the user doesn't say "milestone" or "portfolio" explicitly but is clearly asking about the health of a group of projects, a book of work, or a program. Always use the "Certinia" Salesforce connector for this — it's the source of truth for portfolio, project, milestone, and project task data.
---

# Portfolio Risk Review

Assess the health of every project in a Certinia/PSA portfolio by pulling each project's milestones and project tasks and flagging the ones that are overdue or coming due soon. The output is a ranked risk outline the user can act on.

## Why this shape

A portfolio on its own tells you nothing about risk — the risk lives in the milestones underneath each project. So the workflow always goes portfolio → projects → milestones + project tasks, and the risk judgment is made from the milestone and task dates and status flags, not from the project record alone.

## Workflow

### 1. Resolve the portfolio

Use the **Certinia** connector's `soqlQueryplatform_sobject_all` tool (search for it with `tool_search` if it's not already loaded — the exact tool name is prefixed with the connector name).

​```sql
SELECT Id, Name FROM pse__Portfolio__c WHERE Name LIKE '%<user's portfolio name>%'
​```

- If exactly one match, proceed with it.
- If several match, list them and ask the user which one they meant — don't guess.
- If none match, try a broader LIKE pattern before telling the user it can't be found.

### 2. Find the projects in that portfolio

​```sql
SELECT Id, Name, pse__Stage__c, pse__Is_Active__c
FROM pse__Proj__c
WHERE pse__Portfolio__c = '<portfolioId>'
​```

If this errors because the lookup field is named differently in this org (schemas vary), fall back to `getRelatedRecordsplatform_sobject_all` on the portfolio record, or inspect the object schema, to find the correct relationship field. Don't silently drop this step — the projects list is the backbone of everything that follows.

Only include active projects unless the user asks for a full historical view — closed/completed projects will otherwise dilute the risk picture.

### 3. Run the milestone query action across all projects

Use `pse_MilestoneQueryActionapex_pse_MilestoneQueryAction`. Its `inputs` parameter takes an array, so pass **one entry per project in a single call** rather than calling it once per project — this is faster and keeps the results easy to correlate.

For each project, use an entry like:

​```json
{
  "projectId": "<project Id>",
  "includeStatusAnalysis": true,
  "includeMilestonesTracking": true,
  "includeProjectTasks": true,
  "showOnlyConcerningProjectTasks": false,
  "showOnlyConcerningMilestones": false
}
​```

Leave `showOnlyConcerningMilestones`/`showOnlyConcerningProjectTasks` false on this first pass — you need the full picture (including on-track milestones) to write a fair summary, not just the flagged ones. You can re-query with those flags true later if you need to drill into just one project's problems.

Don't rely on `periodStartDate`/`periodEndDate` to do the risk filtering for you — those only catch milestones starting or ending inside a window, which would miss overdue milestones that ended before the window opened. Instead, pull all milestones and do the date comparison yourself in step 4.

### 4. Run the project task action across all projects

In addition to the milestone query, call the Certinia project task action on the same connector: `pse_ProjectTaskAnalysisActionapex_pse_ProjectTaskAnalysisAction`. If that exact name isn't loaded, `tool_search` for "ProjectTaskAnalysis" to find it. Its `inputs` parameter takes an array, so, as with the milestones, pass **one entry per project in a single call** (using `projectId` for each project) and make the two calls back to back, since they're independent. Check the tool's schema for any additional input flags.

Pull all tasks on this first pass, not just the concerning ones, so you can tell a project with a few late tasks from one with no tasks planned. Use the task results as the source of truth for task-level detail (overdue tasks, blocked or incomplete tasks, end dates versus the milestone's target date). If the milestone action's `includeProjectTasks` output disagrees with the dedicated task action, trust the dedicated task action and mention the discrepancy.

### 5. Classify each project's risk

For each project, look at its milestones' target/end dates and completion status, the `includeStatusAnalysis`/`includeMilestonesTracking` flags returned by the milestone query, and the project tasks returned by the task action, and sort into:

- **At risk (overdue)** — has one or more incomplete milestones whose target date has already passed, or is flagged off-track (actual vs. planned hours variance beyond the built-in threshold), or has incomplete project tasks whose end date has already passed.
- **Watch (due soon)** — no overdue milestones, but has an incomplete milestone due within the next 14 days, or has incomplete project tasks due within the next 14 days, or has project tasks flagged as blocking/incomplete with an end date past the milestone's target.
- **On track** — no overdue or near-term incomplete milestones or project tasks.

14 days is a sensible default lookahead for "due soon" — if the user asks for a different window (e.g. "due this month"), use that instead.

If a project has no milestones or no project tasks at all, say so explicitly rather than omitting it — that's often itself a signal worth surfacing (e.g. a project that hasn't been planned out yet).

### 6. Report the results

Lead with a short prose summary ranking the at-risk projects first and explaining *why* in a sentence or two each (which milestone or task, how overdue, what the hours variance looks like) — this is what the user will actually read first. Follow it with a table for the full picture:

| Project | Risk Level | Overdue Milestones | Overdue Tasks | Due Soon (≤14d) | Key Issue |
|---|---|---|---|---|---|
| ... | At risk / Watch / On track | count | count | count (milestones + tasks) | one-line reason |

Keep the table in the chat response (don't make it a downloadable file) unless the user asks to save/export it.

## Notes

- Tool names in this environment are prefixed with the connector name (e.g. `Certinia:soqlQueryplatform_sobject_all`). If a call fails because the tool isn't loaded, `tool_search` for it first.
- SOQL field names (like the portfolio lookup on `pse__Proj__c`) can vary slightly between Certinia orgs — if a query errors on an unknown field, inspect the object schema rather than guessing a second field name blindly.
- Batch the milestone and project task queries across all projects (one call per action) when possible; only fall back to per-project calls if the batch call errors or the project list is very long.