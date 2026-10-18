<!--
README.md v0.4 — 2026-10-18
Changelog:
- v0.4 (2026-10-18): "at a flat rate" off (Rene: no flat rates); installs are priced per business after a short paid discovery.
- v0.3 (2026-10-14): Spec-only publish (Rene): three-sentence intro, privacy up front, spec link; the quick start returns with the code.
-->

# Capsule Base

Capsule Base lets a business ask its own records questions in plain language and get answers that cite the records they came from. It copies records from the systems the business already runs into one Postgres database on a machine the business owns, indexed for search by meaning and for exact counts, totals and dates. An LLM writes each answer from the records it's given, using your own API key or a local model.

## Your data stays on your machine

- Records live in Postgres on your hardware. Embeddings are computed locally, so loading data sends nothing anywhere.
- When you ask a question, only the question, a description of your tables and the records needed to answer it go to the LLM provider you chose, under your key. Every answer reports what was sent.
- With a local model, nothing leaves the machine.

The full table is in [ARCHITECTURE §9](ARCHITECTURE.md#9-what-leaves-the-machine).

## The spec

This repository publishes the spec first, under the MIT license.

- **[ARCHITECTURE.md](ARCHITECTURE.md)** describes the whole system: the store, ingest, search, the query layer that writes cited answers, security, sizing and limits. It's detailed enough to build a compatible implementation in any language.
- **[connectors/INTERFACE.md](connectors/INTERFACE.md)** is the contract every connector follows: one record envelope and one schema message, sent over HTTP or dropped as files.

The reference implementation, in Python, joins this repository when it's built and tested against the spec. A ten-minute quick start and a setup guide come with it.

## What an answer looks like

```
Q: Which lifts came back with hydraulic problems this year, and did any unit have the problem twice?

A: Three lifts came back with hydraulic problems in 2026. The 19 ft scissor lift A-104 had it
   twice: a weeping lift cylinder when Cedar Flats Paving returned it on May 18 [R1], and low
   fluid with slow lifting when Ridgeline Electric returned it on July 19 [R2]. The 45 ft boom
   A-120 drifted down twice, both times from the jib cylinder seal [R3][R4]. The 26 ft scissor
   lift A-110 had a loose pump fitting in March [R5].

   R1 rentals R-2291 · R2 rentals R-2333 · R3 rentals R-2240 · R4 rentals R-2351 · R5 rentals R-2231
```

The records behind it are the made-up rental company in [examples/](examples/). [ARCHITECTURE §7](ARCHITECTURE.md#7-ask-the-query-layer) shows how the answer is put together: search finds the records that match by meaning, one read-only SQL query handles anything that needs counting or dates, and every claim cites a record.

## Who it's for

- **Builders** who'd rather build it themselves. Read the spec; [§15](ARCHITECTURE.md#15-building-a-compatible-implementation) says what makes an implementation compatible.
- **Almost-builders** who'll get partway and hit the part specific to their system. The [connector chapters](connectors/INTERFACE.md) and [use-case briefs](use-cases/) are written for that part.
- **Owners who want the result without the work.** See [Want it built?](#want-it-built)

## Connectors

| Source | Chapter |
|---|---|
| Anything that exports CSV or Excel | [CSV drop folder](connectors/csv-drop-folder.md) |
| PostgreSQL, SQLite, MySQL | [Direct SQL](connectors/direct-sql.md) |
| QuickBooks Desktop, Access, SQL Server | [ODBC](connectors/odbc.md) |
| Salesforce, HubSpot, Zoho, Xero, FreshBooks, practice-management tools | [REST API](connectors/rest-api.md) |
| FileMaker | [FileMaker](connectors/filemaker.md) |

A connector is any program, in any language, that sends record envelopes. Most are a few hundred lines.

## Use cases

One brief per kind of business: where the data usually lives, which connector fits, and seed questions that show whether it works.

[Equipment rental](use-cases/equipment-rental.md) · [Event production](use-cases/event-production.md) · [Trades and contractors](use-cases/trades-contractor.md) · [Property management](use-cases/property-management.md) · [Veterinary clinic](use-cases/veterinary-clinic.md) · [Tax and accounting](use-cases/tax-accounting.md) · [Law firm](use-cases/law-firm.md)

## What it runs on

A small mini or a VPS with 4 GB of RAM, on macOS or Debian/Ubuntu Linux: PostgreSQL 16+ with pgvector, Ollama for local embeddings, and an Anthropic or OpenAI key or a local chat model. No other outside service. Sizing is in [ARCHITECTURE §12](ARCHITECTURE.md#12-hardware-and-sizing).

## Want it built?

[Emmaus Development](https://emmausdev.com) builds this for businesses on their own hardware, priced per business after a short paid discovery, with a Care Plan that keeps it running as their systems change. Tend8, a paid add-on built on Capsule, adds a morning brief, a live board and alerts; it's also at [emmausdev.com](https://emmausdev.com).

The case for talking to your data instead of clicking through it is made at book length in [The Last Interface](https://last-interface.com).

## License

MIT. See [LICENSE](LICENSE). Use it, change it, build on it, sell setups on it.

Built by [Rene Kropf](https://emmausdev.com) / Emmaus Development LLC.

🐾 [Adopt a Pet — Get a Dog](https://www.petfinder.com/)
