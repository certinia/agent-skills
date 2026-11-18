---
name: certinia-month-end-close
description: Runs month-end close in Certinia Accounting (c2g / FinancialForce) through a Salesforce connector that exposes the c2g actions. Checks period state, sweeps unposted (In Progress) journals, sales invoices, sales credit notes, payable invoices, payable credit notes and cash entries, then posts, discards, cancels or reverses them, and finishes with a close-readiness verdict. Use this whenever the user mentions month end, period close, closing the books, closing a period, unposted or in-progress or draft documents, posting journals, accruals and reversals, cleaning up before close, or asks whether a period is ready to close, even if they never say "c2g", "Certinia" or "FFA".
---

# Month-end close (Certinia c2g)

This skill helps an accountant get a company and period from "open and messy" to "ready to close". It reads a lot, writes carefully, and always says what it did and what is still blocking the close.

It relies on a Salesforce connector, **Certinia** that exposes these Certinia actions: `c2g_Find`, `c2g_PostDocuments`, `c2g_DiscardDocuments`, `c2g_CancelDocuments` and `c2g_ReverseDocuments`, plus a SOQL query tool for reads. Tool names carry a connector-specific prefix; check the tool list to find them. If any of the c2g actions or the query tool is missing, say which, and do only the read-only parts. Verified queries are in `references/queries.md`; read it before writing SOQL.

## Ground rules

**Scope is always one company and one period.** Orgs often have several accounting companies, and period names (such as `2026/010`) repeat across them, so every query and every action filters on company *and* period. Look companies up rather than assuming. Periods are named `YYYY/MMM`; ignore adjustment periods (for example `/100` and `/101`, which are year-end periods dated the last day of the year) unless the user asks for them. If the user hasn't named a period, propose the most recent period that has ended and confirm before doing anything.

**Read first, write second.** Posting creates ledger transactions. Cancel and reverse create new documents. Discard is permanent. None of these can be undone through this connector. So: sweep, triage, show the proposed list, get an explicit yes for that list, then act. A yes to "post these 6" is not a yes to "discard those 2"; confirm each action type separately.

**Batch, then verify.** Every c2g action takes an array, so send all documents for one action in one call. Results come back per document. Report each outcome individually, then re-query the documents' status to confirm what actually happened rather than trusting the response alone.

**Resolve names to IDs with `c2g_Find`.** Users say "JNL000288"; the actions need record IDs. `c2g_Find` wants the exact record name. Documents that came from your own sweep query already carry their Ids.

**Know the limits of the connector.** Month end also involves currency revaluation, allocations, intercompany transfers and eliminations, bank reconciliation, revenue recognition, depreciation, and the period close itself. Use a dedicated tool for any of these only if the connector actually lists one. Otherwise don't improvise with raw record updates, which bypass the accounting engine's own checks; tell the user to run the step in Certinia, then come back and sweep the journals it generated. Never close a period by editing the period record; give a go/no-go verdict and let the user close it in Certinia.

## Workflow

### 1. Scope
Confirm company and period. Fetch the period and report its state: open, soft closed (`c2g__SoftClose__c`, which blocks everything except journals), or closed (`c2g__Closed__c`). If the period is already closed, stop and say so; nothing can be posted to it. Also ask whether the user follows a close checklist or calendar they'd like you to work from; if so, use it as the frame for the summary, but don't assume one exists.

### 2. Unposted-document sweep
For the chosen company, query all six document types for status `In Progress` (queries in the reference file), in parallel. Don't filter to the target period in the query; pull everything In Progress for the company and bucket it in the next step, because stale and misdated documents are exactly what needs surfacing. Journals can also be `Ready To Post`; list those separately, since they are queued rather than posted.

### 3. Triage
Put each In Progress document in one bucket:

| Bucket | Test | Default recommendation |
|---|---|---|
| **Ready to post** | In the target period, date inside the period, has value and lines | Post |
| **Empty** | Journal with `c2g__LineCount__c` = 0, or total of 0 | Discard candidate. Ask; never assume it is junk |
| **Stale** | Period earlier than the target | Flag. Posting back-dates entries into earlier reported numbers. Ask the user what to do |
| **Future-dated** | Period or date after the target | Leave alone, mention it |
| **Period blocked** | Its period is soft closed or closed | Can't post. Say what the user must reopen in Certinia |

Long-lived In Progress documents are common, so many will land in Stale or Future-dated rather than Ready to post. Say so plainly and ask about them; don't offer to post everything just to empty the list.

Journals store credits as negative numbers, so a balanced journal has `Debits + Credits = 0`. If it doesn't, it won't post.

### 4. Act
Show the proposed list for one action: document name, type, date, amount, and what will happen. Wait for an explicit yes. Then call the action.

| Need | Action | Applies to | Notes |
|---|---|---|---|
| Make a draft real | `c2g_PostDocuments` | In Progress: journals, sales invoices, sales credit notes, payable invoices, payable credit notes, cash entries | Posting is synchronous. A very large document can hit limits; post it alone, and if it still fails, ask the user to post it from Certinia, which can run asynchronously |
| Throw a draft away | `c2g_DiscardDocuments` | Same six types, In Progress only | Needs a reason, and the reason must come from the user. Ask for it; don't invent one |
| Undo a posted mistake | `c2g_CancelDocuments` | Complete **journals and cash entries only** | Creates a cancelling document. Sales and payable invoices and credit notes can't be cancelled here; the correction is a credit note, done in Certinia |
| Reverse an accrual | `c2g_ReverseDocuments` | Complete journals only | See the reversal checks below |

**Reverse vs cancel.** Reverse is for entries that are right this month but shouldn't carry forward (accruals, prepayments). Cancel is for entries that were wrong and should vanish. If the user says "reverse" about an error, ask which they mean.

**Reversal checks, before calling `c2g_ReverseDocuments`.** Query the journal first. Skip it and tell the user if `c2g__AutoReverseJournal__c` is true (Certinia already creates and posts the reversal when the journal is posted, so reversing again double-counts), if `c2g__ReversingJournal__c` is already populated (already reversed), or if `c2g__CanceledJournal__c` is true. After reversing, query the returned document and report its period, date and status. The action takes no date or period, so don't assume the reversal lands in the next period; check it, and if it is In Progress, put it through the normal post confirmation.

**When an action fails,** relay the error text exactly as returned, then offer the likely cause (period closed, journal unbalanced or empty, document too large). Don't change a document's date or period to make it post unless the user explicitly asks; that changes which period's numbers move.

### 5. Validate
After acting, confirm the books are sound:
- Re-run the sweep: nothing should remain In Progress in the target period except items the user chose to leave.
- Run the trial-balance integrity check (reference file): net home value for the period must be 0 for each company.
- Look at the journals in the period by type and status (reference file). Recurring month-end work leaves recognisable journals, for example revenue recognition, fixed asset depreciation, currency revaluation, cost amortization, prepaid expenses, and intercompany eliminations (journals owned by an elimination company, `c2g__EliminationCompany__c` = true). If the user expects one of these and none is posted, say so; it may not have been run, or it may be sitting unposted.

### 6. Verdict
End with a go/no-go on closing the period, using the template below. Hard blockers: the period is already closed, any non-zero trial-balance net, documents still In Progress or Ready To Post in the period that the user hasn't chosen to leave, or an expected recurring journal that hasn't been posted. Anything else is a note, not a blocker.

## Output template

Keep the final summary short enough to scan.

```
Month-end readiness: <Company>, <Period>   (as of <date>)

Period: <Open | Soft closed | Closed>

Unposted documents
  <Type>: <n> in period, <n> stale, <n> future, <n> empty
  ...

Ledger checks
  Trial balance net: <0 | amount>
  Journals by type posted this period: <short list>

Done this session: <what was posted/discarded/reversed, with resulting IDs>
Still blocking: <specific items and what resolves each>

Verdict: READY TO CLOSE | NOT READY, <one-line reason>
```

## Style

Talk like a colleague in the finance team: plain, specific, no ceremony. Use document numbers and amounts, not "some documents". When you're unsure what the user intends with an accounting action, ask one question; a wrong post is expensive and a question is cheap.
