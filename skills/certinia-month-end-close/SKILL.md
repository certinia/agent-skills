---
name: "certinia-month-end-close"
description: Runs month-end close in Certinia Accounting (c2g / FinancialForce) through a Salesforce connector that exposes the c2g actions. Checks period state, sweeps unposted (In Progress) journals, sales invoices, sales credit notes, payable invoices, payable credit notes and cash entries, then posts, discards, cancels or reverses them, and finishes with a close-readiness verdict. Use this whenever the user mentions month end, period close, closing the books, closing a period, unposted or in-progress or draft documents, posting journals, accruals and reversals, cleaning up before close, or asks whether a period is ready to close, even if they never say "c2g", "Certinia" or "FFA".
---

# Month-end close (Certinia c2g)

This skill helps an accountant get a company and period from "open and messy" to "ready to close". It reads a lot, writes carefully, and always says what it did and what is still blocking the close.

It relies on a Salesforce connector that exposes these Certinia actions: `c2g_Find`, `c2g_PostDocuments`, `c2g_DiscardDocuments`, `c2g_CancelDocuments` and `c2g_ReverseDocuments`, plus a SOQL query tool for reads. Tool names carry a connector-specific prefix; check the tool list to find them. If any of the c2g actions or the query tool is missing, say which, and do only the read-only parts. Verified queries are in the **Queries** section at the end of this file; use them before writing your own SOQL.

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
For the chosen company, query all six document types for status `In Progress` (see Queries: unposted sweeps), in parallel. Don't filter to the target period in the query; pull everything In Progress for the company and bucket it in the next step, because stale and misdated documents are exactly what needs surfacing. Journals can also be `Ready To Post`; list those separately, since they are queued rather than posted.

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
- Run the trial-balance integrity check (see Queries): net home value for the period must be 0 for each company.
- Look at the journals in the period by type and status (see Queries). Recurring month-end work leaves recognisable journals, for example revenue recognition, fixed asset depreciation, currency revaluation, cost amortization, prepaid expenses, and intercompany eliminations (journals owned by an elimination company, `c2g__EliminationCompany__c` = true). If the user expects one of these and none is posted, say so; it may not have been run, or it may be sitting unposted.

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

## Queries

These use standard Certinia Accounting objects and fields only. Run them with the connector's SOQL query tool. Replace `{Company}` and `{YYYY/MMM}` with real values. Always filter by company as well as period, because period names repeat across companies. If a query fails on a field, run a schema describe on that object and adapt; field availability can vary with package version and enabled features.

### Companies and periods

```sql
SELECT Id, Name, c2g__EliminationCompany__c FROM c2g__codaCompany__c
```

```sql
SELECT Id, Name, c2g__StartDate__c, c2g__EndDate__c, c2g__Closed__c, c2g__SoftClose__c,
       c2g__AR__c, c2g__AP__c, c2g__Cash__c
FROM c2g__codaPeriod__c
WHERE c2g__OwnerCompany__r.Name = '{Company}' AND c2g__EndDate__c <= {YYYY-MM-DD}
ORDER BY c2g__EndDate__c DESC LIMIT 6
```

`c2g__Closed__c` is final close; `c2g__SoftClose__c` blocks everything except journals. `c2g__AR__c`, `c2g__AP__c` and `c2g__Cash__c` close individual GLA groups.

### Unposted sweeps (run all six in parallel)

Status is `In Progress` for every type; the status field name differs by type. None of these filter on period, on purpose: triage buckets by period.

Journals:
```sql
SELECT Id, Name, c2g__Type__c, c2g__JournalStatus__c, c2g__JournalDate__c, c2g__Period__r.Name,
       c2g__JournalDescription__c, c2g__Debits__c, c2g__Credits__c, c2g__LineCount__c,
       c2g__AutoReverseJournal__c
FROM c2g__codaJournal__c
WHERE c2g__JournalStatus__c IN ('In Progress','Ready To Post') AND c2g__OwnerCompany__r.Name = '{Company}'
ORDER BY c2g__JournalDate__c DESC
```
Credits are stored negative. Journals with `c2g__LineCount__c` = 0 are empty shells. Only `In Progress` journals can be posted or discarded with the c2g actions; `Ready To Post` ones are queued for posting and should be reported, not acted on.

Sales invoices:
```sql
SELECT Id, Name, c2g__InvoiceDate__c, c2g__InvoiceTotal__c, c2g__Account__r.Name, c2g__Period__r.Name
FROM c2g__codaInvoice__c
WHERE c2g__InvoiceStatus__c = 'In Progress' AND c2g__OwnerCompany__r.Name = '{Company}'
```

Sales credit notes:
```sql
SELECT Id, Name, c2g__CreditNoteDate__c, c2g__CreditNoteTotal__c, c2g__Account__r.Name, c2g__Period__r.Name
FROM c2g__codaCreditNote__c
WHERE c2g__CreditNoteStatus__c = 'In Progress' AND c2g__OwnerCompany__r.Name = '{Company}'
```

Payable invoices:
```sql
SELECT Id, Name, c2g__InvoiceDate__c, c2g__InvoiceTotal__c, c2g__AccountInvoiceNumber__c,
       c2g__Account__r.Name, c2g__Period__r.Name
FROM c2g__codaPurchaseInvoice__c
WHERE c2g__InvoiceStatus__c = 'In Progress' AND c2g__OwnerCompany__r.Name = '{Company}'
```

Payable credit notes:
```sql
SELECT Id, Name, c2g__CreditNoteDate__c, c2g__CreditNoteTotal__c, c2g__Account__r.Name, c2g__Period__r.Name
FROM c2g__codaPurchaseCreditNote__c
WHERE c2g__CreditNoteStatus__c = 'In Progress' AND c2g__OwnerCompany__r.Name = '{Company}'
```

Cash entries:
```sql
SELECT Id, Name, c2g__Type__c, c2g__Date__c, c2g__Reference__c, c2g__Period__r.Name
FROM c2g__codaCashEntry__c
WHERE c2g__Status__c = 'In Progress' AND c2g__OwnerCompany__r.Name = '{Company}'
```
This query has no amount; if the user needs amounts, query `c2g__codaCashEntryLineItem__c` for the entry. Documents dated in the future do occur, so compare each document's period and date against the target period rather than assuming everything In Progress belongs to it.

### Reversal pre-check

Run this on the journal Ids before calling `c2g_ReverseDocuments`:
```sql
SELECT Id, Name, c2g__JournalStatus__c, c2g__Type__c, c2g__AutoReverseJournal__c,
       c2g__ReversingJournal__c, c2g__CanceledJournal__c, c2g__Period__r.Name, c2g__JournalDate__c
FROM c2g__codaJournal__c WHERE Id IN ('{id1}','{id2}')
```
Safe to reverse only when status is Complete, `AutoReverseJournal` is false, `ReversingJournal` is null and `CanceledJournal` is false.

After reversing, check the new document the same way, using the Id the action returned. `c2g__SourceJournal__c` and `c2g__OriginalJournal__c` link reversing and cancelling journals back to their source.

### Trial balance integrity check

Net home value of all transaction lines in a period must be zero, per company:
```sql
SELECT c2g__OwnerCompany__r.Name co, SUM(c2g__HomeValue__c) net, COUNT(Id) n
FROM c2g__codaTransactionLineItem__c
WHERE c2g__Transaction__r.c2g__Period__r.Name = '{YYYY/MMM}'
GROUP BY c2g__OwnerCompany__r.Name
```
Any non-zero `net` for a company is a hard blocker. Treat floating-point noise (for example 1e-12) as zero. A period with no transactions returns no rows, which is not a pass; report "no activity" instead.

### Journals by type

```sql
SELECT c2g__Type__c, c2g__JournalStatus__c, COUNT(Id) n, SUM(c2g__Debits__c) debits
FROM c2g__codaJournal__c
WHERE c2g__Period__r.Name = '{YYYY/MMM}' AND c2g__OwnerCompany__r.Name = '{Company}'
GROUP BY c2g__Type__c, c2g__JournalStatus__c
```
Standard journal types include Manual Journal, Revenue Recognition Journal, Revenue Recognition True Up Journal, Fixed Asset Acquisition / Depreciation / Disposal Journal, Currency Revaluation Journal, Cost Amortization Journal, Prepaid Expenses Journal, Income Schedule, Reversing Journal, Cancelling Journal, Cash Matching Journal and Year End Journal. Which of these exist depends on the Certinia products installed. Journal statuses: In Progress, Ready To Post, Complete, Discarded.

### Notes

- `c2g_Find` takes exact record names (for example a journal `JNL000288` or sales invoice `SIN000444`) and returns the Id. Number formats vary by org configuration.
- Sweep queries can return many rows in a busy company. Add a `LIMIT` if needed and tell the user the list is truncated.
