<!--
law-firm.md v0.1 — 2026-10-14
Changelog:
- v0.1 (2026-10-14): First draft.
-->

# Use case: small law firm

A small firm's memory is its matters, time entries, invoices and docket, plus whatever the associate who left knew. "How many hours did we bill on custody cases last year, and what did we actually collect?" and "Which matters have gone quiet for 60 days?" are questions the practice-management system holds the data for and doesn't answer. An hour saved finding information is an hour that can be billed.

Read the confidentiality section below before loading anything.

## Where the data lives

- **Practice management:** matters, clients, matter types, responsible attorneys, open and close dates, notes. Cloud or on-premises.
- **Time and billing:** time entries with narratives, invoices, payments, write-downs.
- **Docket and calendar:** deadlines, hearings, filings.
- **Document management:** document metadata (title, date, matter, author). Load metadata, not documents.

## Connector

- [REST API](../connectors/rest-api.md) for cloud practice-management tools.
- [ODBC](../connectors/odbc.md) or [direct SQL](../connectors/direct-sql.md) for on-premises systems.
- [CSV drop folder](../connectors/csv-drop-folder.md) for billing and docket reports.

## Load first

| Table | Key | Title | Why |
|---|---|---|---|
| matters | matter number | client + matter name | Type, status, attorney, dates |
| time entries | entry id | timekeeper + matter + date | Hours, rates, narratives |
| invoices and payments | invoice number | client + matter | Billed, collected, written down |
| docket events | event id | matter + event | Last activity, upcoming deadlines |
| matter notes | note id | matter + date | Strategy and outcomes |

## Seed questions

**What happened:**

- What have we argued about relocation in past custody matters, and how did it turn out?
- Which matters had opposing counsel from Harrow & Vance, and how were they resolved?

**How many, how much, when:**

- How many hours did we bill on custody matters last year, and what was the effective rate (collected divided by hours billed)?
- Which open matters have had no time entries or docket activity in 60 days?
- Which clients have invoices unpaid for more than 90 days?
- What's the average time from intake to settlement, by matter type?
- What were write-downs by timekeeper this year?

**Both:**

- Which open matters have gone quiet for 60 days, and what was the last thing done on each?

## Descriptions that matter

- Matter types and status codes.
- Billed hours versus worked hours; billed versus collected amounts.
- Which date marks a matter's last activity.

## Confidentiality

- **Client confidences.** Sending matter data to a cloud LLM provider puts it under that provider's data terms. Check retention and training terms against your duties before choosing one. ABA Formal Opinion 512 (2024) covers lawyers' duties when using generative AI tools. A local model (`AI_PROVIDER=ollama`) keeps everything on the machine.
- **Ethical walls.** Capsule has no per-user permissions. Everyone with the token can ask about every matter loaded. Leave walled matters out, or run a separate instance for them.
- **Conflicts.** Capsule can help search past matters and parties. It doesn't replace the firm's conflict-check process.
