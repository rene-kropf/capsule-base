<!--
property-management.md v0.1 — 2026-10-14
Changelog:
- v0.1 (2026-10-14): First draft.
-->

# Use case: property management

Property-management software holds tenants, leases, work orders and vendor bills, and it runs the reports it was built to run. The questions that drive capital decisions fall between them: which units keep having the same problem, what a turnover really costs, which vendor is slow. The business is already paying for software that can't answer those questions.

## Where the data lives

- **Property-management software:** properties, units, leases, tenants, rent charges and payments, work orders. Usually cloud.
- **Accounting:** vendor bills and payments, if not in the property software.
- **Your own spreadsheets:** capex plans, market rent comparisons.

## Connector

- Scheduled report exports into the [CSV drop folder](../connectors/csv-drop-folder.md). Most property platforms can email or save reports on a schedule.
- [REST API](../connectors/rest-api.md) where the vendor offers one. API access varies a lot in this market; ask the vendor first.

## Load first

| Table | Key | Title | Why |
|---|---|---|---|
| units | unit id | property + unit | Everything hangs off the unit |
| leases | lease id | unit + tenant | Terms, rent, start and end dates |
| work orders | work order number | unit + problem | What broke, who fixed it, how long it took |
| vendor bills | bill number | vendor + unit | What repairs cost |
| rent ledger | charge or payment id | unit + month | Late payments, balances |
| market comps (your spreadsheet) | your id | unit type + area | Only if you keep one |

## Seed questions

**What happened:**

- Which units have had plumbing problems more than twice, and what was done each time?
- What did tenants report about heating at the 14th Street building last winter?

**How many, how much, when:**

- Which leases end in the next 90 days, and what's the current rent on each?
- What did we spend per unit with each trade this year?
- What's the average number of days from work order opened to completed, by vendor?
- Which tenants paid late three or more times in the last 12 months?
- What did make-ready work cost per unit turnover this year?
- Which properties have the highest maintenance cost per unit?

**Both:**

- Which leases end in the next 90 days, how does each rent compare with our comps sheet, and what's the work-order history of those units?

Capsule only knows what's loaded. Comparing rents with the market needs a comps table, from your own spreadsheet or a data service you already pay for.

## Descriptions that matter

- Work-order categories and status codes.
- Rent amount versus charges (rent, fees, utilities) in the ledger.
- Lease dates: start, end, move-in, move-out, notice given.

## Watch for

- **Application data.** Leave out Social Security numbers, income documents and bank details. Operations questions don't need them.
- **Free-text work orders.** "Leak under sink" and "kitchen plumbing" are the same problem. Search by meaning handles that for "what happened" questions. For counts, a category field is more reliable than the description.
