<!--
equipment-rental.md v0.2 — 2026-10-14
Changelog:
- v0.2 (2026-10-14): Spec-only publish wording.
- v0.1 (2026-10-14): First draft.
-->

# Use case: equipment rental

A rental business runs on contracts, check-in notes, service tickets and the counter staff's memory. Which units keep coming back damaged, which customers run late, what a unit earned against what it cost to fix: all of it is in the records, and nobody has the hours to pull it. The examples in the spec use a made-up rental company (`examples/sample_rental/`), so this brief lines up with them.

## Where the data lives

- **Rental management software:** customers, fleet, quotes, contracts and contract lines, returns.
- **Shop or service records:** repair tickets, parts, hours. Sometimes in the rental system, often a spreadsheet.
- **Accounting:** invoices and payments, if they aren't in the rental system.

## Connector

- Scheduled exports from the rental system into the [CSV drop folder](../connectors/csv-drop-folder.md). Start here.
- [ODBC](../connectors/odbc.md) or [direct SQL](../connectors/direct-sql.md) when the system runs on an on-site database you're allowed to read.
- [REST API](../connectors/rest-api.md) for cloud systems that offer one.
- [FileMaker](../connectors/filemaker.md) for custom-built systems.

## Load first

| Table | Key | Title | Why |
|---|---|---|---|
| rentals (contract lines) | line or contract number | item name | The center of almost every question |
| items (fleet) | asset tag | item name | Categories, rates, purchase dates |
| customers | account number | name | Names, types, credit terms |
| service tickets | ticket number | item name + problem | Repair cost and what the shop found |
| quotes | quote number | customer + items | Which quotes turned into rentals |

Load contract **lines**, one row per unit out, not only contract headers. Questions are about units.

## Seed questions

**What happened** (search by meaning carries these):

- Which units came back with hydraulic problems this year, and who had them out?
- What did the shop find on the last three repairs to the 60 ft boom?
- Which customers have notes about late returns or billing disputes?

**How many, how much, when** (the SQL views carry these):

- Which units are out past their due date right now?
- What did each equipment category earn last quarter, and how many days was it out?
- Which units cost more in repairs this year than they earned in rent?
- Which quotes from last month didn't turn into rentals?
- Who were our top ten customers by revenue this year, and how does that compare with last year?
- How many generators were out on our busiest day in September?

**Both:**

- Which customers rented compactors and came back with damage notes, and what did we charge them?

## Descriptions that matter

- What the billed amount includes: damage, cleaning, fuel, delivery, tax.
- Day, week and month rates, and which one a contract used.
- Status codes on contracts and units (out, returned, in shop, sold).
- Which date is which: out, due, returned, checked in, invoiced.

## Watch for

- **Units sharing a model name.** Two "19 ft scissor lift" units are different assets. Key on the asset tag and send it in every rental.
- **Late check-ins.** If the counter enters returns a day or two after the unit is back, "returned date" means "date entered". Say so in its description.
- **Multi-item contracts.** A contract total split across lines is easy to double-count. Describe which amount lives where.
