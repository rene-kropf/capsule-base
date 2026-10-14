<!--
trades-contractor.md v0.1 — 2026-10-14
Changelog:
- v0.1 (2026-10-14): First draft.
-->

# Use case: trades and contractors

Electricians, plumbers and HVAC shops estimate from memory. The job history that would make the next bid accurate is in the field-service app and the accounting system: what a similar job really took, what went wrong, what got called back. One accurate bid a month is worth more than the setup.

## Where the data lives

- **Field-service software:** customers, sites, jobs, estimates, tech notes, time entries. Usually cloud.
- **Accounting:** QuickBooks Desktop or Online, Xero. Invoices, line items, materials.
- **Price book and supplier invoices:** often spreadsheets or emailed CSVs.

## Connector

- [REST API](../connectors/rest-api.md) for cloud field-service apps, or their scheduled CSV exports into the [drop folder](../connectors/csv-drop-folder.md).
- QuickBooks Desktop through [ODBC](../connectors/odbc.md); QuickBooks Online and Xero through [REST](../connectors/rest-api.md).
- The [direct SQL](../connectors/direct-sql.md) chapter's example script is a field-service `jobs` table.

## Load first

| Table | Key | Title | Why |
|---|---|---|---|
| jobs | job number | job summary | Type, dates, estimate, actuals, tech notes |
| estimates | estimate number | customer + scope | What was quoted, and whether it was won |
| invoice lines | invoice + line | item | Materials and labor actually billed |
| time entries | entry id | tech + job + date | Labor hours per job and tech |
| customers and sites | customer or site id | name or address | Repeat work, site history |

## Seed questions

**What happened:**

- What did we find on past jobs at 42 Larkspur Lane?
- What went wrong on our 200 A panel upgrades last year?
- Which jobs had a callback within 30 days, and what caused it?

**How many, how much, when:**

- What did we charge for 200 A panel upgrades last year: lowest, highest, typical?
- Which jobs went over estimate last quarter, and by how much?
- What's the average labor time on a water heater replacement, by tech?
- Which estimates are more than 14 days old with no follow-up?
- How has material cost per job type changed from last year to this year?
- Which maintenance-agreement customers haven't had a visit in 12 months?

**Both:**

- Which jobs went over estimate last quarter, and what do the tech notes say about why?

## Descriptions that matter

- Estimate amount versus invoiced amount; whether either includes tax or permits.
- Hours worked versus hours billed.
- Job types. If they're typed freehand ("panel upg", "200A service"), say so, or clean them into a fixed list in the connector.

## Watch for

- **Customer versus site.** A landlord with twelve properties is one customer and twelve sites. Send both names on every job.
- **Two systems, two names.** QuickBooks items and field-service job types rarely match. Join them in the connector or describe how they relate.
