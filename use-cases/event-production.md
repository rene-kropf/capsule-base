<!--
event-production.md v0.1 — 2026-10-14
Changelog:
- v0.1 (2026-10-14): First draft.
-->

# Use case: event production

A production company lives on shows: quotes, gear lists, crew calls, load-ins, invoices. The questions that matter cut across all of them. Did we make money on that show? Who worked it? What broke? Will that client book again? Today the answers are spread over a show-management tool, a crew spreadsheet, the gear log and the accounting system.

## Where the data lives

- **Show or project management:** shows, venues, quotes, gear lists.
- **Crew scheduling:** often a spreadsheet or a scheduling app. One row per person per call.
- **Gear inventory and repair log.**
- **Accounting:** invoices, payments, labor costs.

## Connector

- [CSV drop folder](../connectors/csv-drop-folder.md) for spreadsheets and exports. Most production companies start here.
- [REST API](../connectors/rest-api.md) for cloud show-management, CRM and scheduling tools.
- QuickBooks through [ODBC](../connectors/odbc.md) (Desktop) or [REST](../connectors/rest-api.md) (Online).

## Load first

| Table | Key | Title | Why |
|---|---|---|---|
| shows | show id | show name + date | The center of every question |
| quotes | quote id + version | show name | What was promised |
| invoices | invoice number | client + show | What was billed |
| crew calls | call id | person + show + date | Who worked, when, how long |
| gear out / gear in | checkout id | item + show | What went where and how it came back |
| clients | client id | name | Rebooking and history |

## Seed questions

**What happened:**

- What went wrong on load-in at outdoor shows last season?
- What notes do we have from the last time we worked the Riverside Amphitheater?
- Which gear came back damaged from outdoor events, and what was wrong with it?

**How many, how much, when:**

- Which shows ran over on labor hours, and by how much against the quote?
- What did we bill for the spring gala against what we quoted?
- Who worked the most weekend calls this quarter?
- Which clients booked in March last year and haven't booked yet this year?
- How many crew hours did we schedule each week in October?

**Both:**

- Which clients complained about audio last year, and how much did we bill each of them?

## Descriptions that matter

- Call time versus hours worked versus hours billed.
- Show date versus invoice date versus payment date.
- Quote versions: which one was accepted. Send only accepted versions, or a field that marks them.

## Watch for

- **Crew personal data.** Leave pay rates, phone numbers and addresses out unless someone needs to ask about them.
- **Hand-edited spreadsheets.** Rows get deleted and re-added. Load those tables as snapshots.
- **One venue, five spellings.** Add a venues table, or clean venue names in the connector, before asking venue questions.
