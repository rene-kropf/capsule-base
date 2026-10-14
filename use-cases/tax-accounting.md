<!--
tax-accounting.md v0.1 — 2026-10-14
Changelog:
- v0.1 (2026-10-14): First draft.
-->

# Use case: tax and accounting practice

A small firm carries years of client history: filing records, deduction patterns, entity structures, estimated payments, notes from every conversation. Today, comparing five years of one client's charitable giving means opening five returns. Flagging every client whose estimates are behind pace means a spreadsheet someone rebuilds each quarter.

Read the confidentiality section below before loading anything.

## Where the data lives

- **Tax preparation software:** clients, returns by year and form, key amounts. Export options vary; many products offer client lists and return summary reports.
- **Practice management:** engagements, deadlines, extensions, workflow status, time, notes. Often cloud with an API.
- **Bookkeeping clients' ledgers:** QuickBooks or Xero, when the firm keeps their books.

## Connector

- [CSV drop folder](../connectors/csv-drop-folder.md) for tax software reports.
- [REST API](../connectors/rest-api.md) for cloud practice-management tools, QuickBooks Online and Xero.
- [ODBC](../connectors/odbc.md) for on-premises tax or practice software with a database you're allowed to read.

## Load first

| Table | Key | Title | Why |
|---|---|---|---|
| clients and entities | client id | name | Entity type, relationships, preparer |
| return summaries | client + year + form | client + year | AGI, deductions, credits, balance due: the numbers people compare |
| estimated payments | payment id | client + quarter | Pace against last year |
| deadlines and extensions | client + year + form | client + form | What's due, what's filed |
| engagement notes | note id | client + date + subject | Decisions and their reasons |

## Seed questions

**What happened:**

- What did we decide about the Johnsons' home office deduction in past years, and why?
- Which clients had an IRS or state notice in the last two years, and how was it resolved?

**How many, how much, when:**

- What were the Johnsons' charitable contributions for each of the last five years?
- Which clients' estimated payments this year are behind last year's pace?
- Which returns are on extension and not started, and when is each due?
- Which clients received K-1s from more than three partnerships last year?
- Which clients' AGI changed by more than 30% from the prior year?
- What were average hours per partnership return by preparer this season against last?

**Both:**

- Which clients are behind on estimates this year, and what do our notes say about their situation?

## Descriptions that matter

- Which amounts come from which line or form, and the year they belong to.
- Entity types and how related entities link (owner, spouse, partnership, trust).
- Workflow status codes.

## Confidentiality

- **Tax return information has its own disclosure rules.** In the US, IRC §7216 limits disclosure of tax return information without client consent. Sending return data to a cloud LLM provider may count as a disclosure. Get advice from your professional liability carrier or counsel before using a cloud provider, or run a local model (`AI_PROVIDER=ollama`), where nothing leaves the machine.
- **Leave identifiers out.** Social Security numbers, EINs and bank account numbers aren't needed to answer practice questions. Drop them in the connector.
- **One trust boundary.** Everyone with the Capsule token can ask about every client. If some staff shouldn't see some clients, run separate instances.
