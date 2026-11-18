---
name: csc-onboarding-kickoff
description: Guided customer onboarding kickoff for Certinia Customer Success (CSC). Takes a closed-won opportunity and walks through creating the success plan, objectives, KPIs, an onboarding playbook from a template, resource allocation, and task assignments, with a confirmation before anything is saved. Use this whenever the user wants to kick off onboarding, start a new customer, set up a success plan from a won deal, build onboarding objectives or KPIs, spin up an onboarding playbook, or assign onboarding tasks, even if they never say "kickoff" or name the individual steps.
---

# CSC Onboarding Kickoff

Turn a won opportunity into a ready-to-run onboarding: a success plan with objectives and KPIs, an onboarding playbook with allocated resources, and tasks with owners. The steps depend on each other (each one produces IDs the next needs), which is why this is a guided sequence instead of a set of separate tool calls.

This skill is written to work in any customer org. Object names, templates, picklist values, and licensed features differ between orgs, so discover them at runtime instead of assuming them.

## Ground rules

- **Nothing is created without the user's confirmation.** Several actions have a `Verify` mode that returns a draft and saves nothing. Use it first, show the draft, and only re-run with `Save` after the user agrees. For actions with no preview mode, summarize the exact inputs and ask before calling.
- **Never invent IDs.** Opportunity, account, template, objective, playbook, task, and resource IDs all come from the user or from a query result.
- **Record IDs between steps.** Keep a running list of the IDs you create so you can pass them forward and list them in the final summary.
- **Handle gaps gracefully.** If a step cannot run (no template, feature not licensed, no matching resources), say so plainly, skip it, and continue with the steps that still make sense.

## Step 0: Check what's available

Use tool search to confirm the **Certinia** connector exposes these actions. Only the first is strictly required; the rest are optional steps.

| Step | Action | Required? |
|---|---|---|
| Success plan | `csc_ActionGenAiSuccessPlanFromOpp` (AI) or `csc_ActionCSPFO` (manual) | one of the two |
| Objectives | `csc_ActionGenAiGenerateObjectives` | optional |
| KPIs | `csc_ActionGenerateObjectiveKPIs` | optional |
| Playbook | `csc_ActionCPFT` | optional |
| Allocation | `csc_ActionAutoAllocate` | optional |
| Task assignment | `csc_AssignPlaybookTasksAction` | optional |

If none of the success plan actions are available, stop and tell the user this connector doesn't appear to have Certinia CS actions enabled. If some optional ones are missing, tell the user which steps will be skipped before starting.

Also locate the connector's generic query and schema tools (SOQL query, search, object schema). You need them to find opportunities, templates, and the records created along the way.

## Step 1: Identify the opportunity

Ask for the opportunity if the user hasn't given one (name, ID, or account). Search for it and confirm:

- It exists and is the one they mean (show name, account, amount, stage, close date).
- It is Closed Won. If not, warn the user and ask whether to continue anyway.
- No success plan already exists for it. Query for existing success plans on that account or opportunity first. If one exists, show it and ask whether to extend it (skip to Step 3) or create another. This prevents duplicate plans when the skill is run twice.

Collect: `OpportunityId`, `AccountId`, and a plan name (suggest "<Account> Onboarding" and let the user change it).

## Step 2: Create the success plan

Offer the user the two routes and recommend the AI route when it's available:

**AI route.** Call `csc_ActionGenAiSuccessPlanFromOpp` with `Mode: "Verify"` and the `OpportunityId`. Show the generated summary and challenges/risks. If the user approves (or edits), call it again with `Mode: "Save"` and a `PlanName`. The plan name is required in Save mode.

**Manual route.** Call `csc_ActionCSPFO` with `OpportunityId` and `SuccessPlanName` (both required). Optionally pass `AccountId`, `Summary`, `EffectiveDate`, and `OwnerId`. Use this when AI features aren't enabled or the user wants to write their own summary.

Save actions may not return the new plan's ID in an obvious form. If it isn't returned, query for the success plan on the opportunity or account to get it. Record `SuccessPlanId`.

## Step 3: Objectives

Call `csc_ActionGenAiGenerateObjectives` with the `SuccessPlanId` and `Mode: "Verify"`. Present the proposed objectives as a short list. Ask the user to approve, drop, or reword them, then call again with `Mode: "Save"`.

If the user wants specific objectives rather than AI-generated ones, tell them to add those in the app, or offer to create objectives from a template if `csc_ActionCOFT` is available.

After saving, query the objectives under the plan to get their IDs. Record `ObjectiveIds`.

## Step 4: KPIs

Call `csc_ActionGenerateObjectiveKPIs` with the `SuccessPlanId` and the `ObjectiveIds` (an array), `Mode: "Verify"`. Show the KPIs grouped by objective. After confirmation, call again with `Mode: "Save"`.

Let the user choose which objectives get KPIs. Not every objective needs them.

## Step 5: Onboarding playbook from a template

1. Find candidate playbook templates by querying for playbooks flagged as templates. Use the schema tool to find the exact object and field names in this org instead of guessing.
2. Show the options and ask which template to use. If there is exactly one obvious onboarding template, recommend it. If there are none, skip this step and the two after it, and tell the user they need a template playbook first.
3. Ask for the start date (default to today, or the plan's effective date if one is set) and an owner if relevant.
4. Confirm, then call `csc_ActionCPFT` with:
   - `TemplatePlaybookId`, `PlaybookName`, `StartDate`: required.
   - `IsTemplate: false` and `IsActive: true`: both required, and this is a live playbook, not a new template.
   - `ParentSuccessPlanId`, `AccountId`, and `ObjectiveId` (when the playbook belongs under a specific objective).
   - `IsAsynchronousProcess: false`. The next steps need the playbook's tasks to exist, and the default asynchronous mode creates them in the background, so they may not be there yet.
   - `CopySkillsFromPlaybook: true` if the template has skills, since allocation uses skills to match resources.

Record `PlaybookId`. Playbook start date shifts all related task dates, so double-check the date with the user.

## Step 6: Allocate resources

Call `csc_ActionAutoAllocate` with the `PlaybookId`. It uses the org's configured allocation method to pick best-matched resources.

After it runs, query the playbook's tasks and their assigned resources, then report what was allocated and what's still unassigned. If nothing was allocated, the likely causes are missing skills on the template, no resources with matching skills, or allocation not configured. Say which seem likely and move on to Step 7 for manual assignment.

## Step 7: Assign remaining tasks

For tasks still without an owner, ask the user who should take them. Look up resources by name with a query rather than asking for IDs.

`csc_AssignPlaybookTasksAction` takes a `PlaybookTaskId` and a list of `ResourceIds` and **sets the task's resources to exactly that list**. Resources not on the list are removed. So:

- Before calling, read the task's current assignees.
- Never use it on a task that already has the right people unless the user wants a change.
- Show the proposed before and after for each task and get a confirmation.

Batch tasks that share an assignee into one call.

## Step 8: Wrap up

Finish with a short summary:

- What was created, with names and record IDs (success plan, objectives, KPIs, playbook).
- Allocation results: who is on which tasks, and anything still unassigned.
- Steps that were skipped and why.
- Suggested next steps: schedule the kickoff meeting, share the plan with the customer, review the KPIs' targets.

Keep it scannable. Link to records if the connector gives record URLs, otherwise give the IDs.

## Handling problems

- **An action errors.** Show the error message in plain language and say which earlier steps succeeded so the user knows the current state. Don't retry blindly. If a retry could create duplicates (Save modes, CPFT), check whether the record was created first.
- **Permission or license errors.** Name the step and the likely cause, skip it, and continue.
- **The user wants to stop partway.** Summarize what exists so far. Everything saved remains in the org, so be explicit about that.
- **The user wants to undo something.** Deletion is a separate decision that needs explicit confirmation of exactly which records. Offer it only if the user asks.
