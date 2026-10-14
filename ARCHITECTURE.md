<!--
ARCHITECTURE.md v0.4 — 2026-10-14
Changelog:
- v0.4 (2026-10-14): §8 OPENAI_BASE_URL, so any OpenAI-compatible endpoint (RouterLM, a local engine) can answer; §9 RouterLM line matches its spec.
- v0.3 (2026-10-14): Spec-only publish (Rene): reference implementation joins later; SETUP.md references removed.
- v0.2 (2026-10-14): §7.2: questions about what's missing go to SQL. Spec approved by Rene.
- v0.1 (2026-10-14): First draft of the spec, written against the release code (see staging GAPS.md).
-->

# Capsule Base: architecture

Capsule Base copies the records a business already keeps into one Postgres database on a machine the business owns. Then it answers plain-language questions about those records and cites the records each answer came from.

This document is the spec. Its reference implementation, in Python, joins this repository when it's built and tested against this document. File names in this document are that implementation's. A section marked *optional* describes something a conforming build may leave out.

## Contents

1. [What it does](#1-what-it-does)
2. [Design rules](#2-design-rules)
3. [The pieces](#3-the-pieces)
4. [The store](#4-the-store)
5. [Ingest](#5-ingest)
6. [Search](#6-search)
7. [Ask: the query layer](#7-ask-the-query-layer)
8. [LLM providers](#8-llm-providers)
9. [What leaves the machine](#9-what-leaves-the-machine)
10. [Interfaces](#10-interfaces)
11. [Security](#11-security)
12. [Hardware and sizing](#12-hardware-and-sizing)
13. [Freshness](#13-freshness)
14. [Limits](#14-limits)
15. [Building a compatible implementation](#15-building-a-compatible-implementation)

---

## 1. What it does

A rental company wants to know which lifts came back with hydraulic problems this year and who had them out. A law firm wants the matters with no activity in 60 days. A clinic wants every dog over eight seen for limping in the last two years and what was found. The answers are already in their systems. Getting them out today means a report someone has to build, or an afternoon of clicking.

Capsule Base does three things:

1. **Copies records in.** Small programs called connectors read the source systems (CSV exports, SQL databases, ODBC drivers, REST APIs, FileMaker) and send each record to Capsule in one standard envelope.
2. **Indexes them two ways.** Each record is rendered as text and embedded, so it can be found by meaning. Each record's fields also show up in typed, read-only SQL views, so the questions that need counting, summing and date math can be answered exactly.
3. **Answers questions.** A question goes in. Capsule finds the relevant records and runs a read-only query if the question needs one. The model writes an answer from what came back, and every claim cites a record by its key in the source system.

## 2. Design rules

These hold for every part of the system. When a design choice is close, these decide it.

- **One database.** Postgres holds the records, the vectors, the field schema and the query views. There is no second store, no queue and no cache server.
- **The source system stays the system of record.** Capsule keeps a read-only copy. It never writes back to a source, and its API doesn't edit records that came from one.
- **Connectors speak one envelope.** A connector is any program, in any language, that produces record envelopes (see [connectors/INTERFACE.md](connectors/INTERFACE.md)). The core doesn't import vendor SDKs, so when a vendor changes its API, only that one connector needs fixing.
- **Embeddings run locally.** Ingest never sends record content to an LLM provider.
- **At question time, the least data that answers the question leaves.** The question, the schema description and the records needed for the answer go to the LLM provider you configured, under your own key. With a local model, nothing leaves the machine.
- **Every claim points to a record.** If the records don't hold the answer, the answer says so.
- **Nothing is silently deleted.** A record deleted at the source is marked deleted in Capsule and drops out of answers. The row stays until a person removes it.

## 3. The pieces

```
 source systems         connectors                 Capsule (one machine)
 ──────────────         ──────────                 ─────────────────────────────────────────
 CSV / Excel exports ─► drop folder ──────┐
 Postgres, SQLite    ─► sql script  ──────┤
 ODBC sources        ─► odbc script ──────┼─► POST /ingest ─► entries ─► chunk ─► embed ─► embeddings
 REST APIs           ─► api script  ──────┤                     │        (local model, Ollama)
 FileMaker           ─► export script ─► drop folder            └─► data.* typed views (read-only)

 question ─► POST /ask ─► 1 retrieve ─► 2 plan ─► 3 query ─► 4 answer ─► answer + citations
                          (hybrid       (LLM:     (read-only (LLM)
                           search)      SQL or    SQL)
                                        none)
```

| Piece | What it is | In the reference implementation |
|---|---|---|
| Store | PostgreSQL 16+ with pgvector 0.5+ | `models.py`, `migrations/` |
| API server | FastAPI app, one process | `main.py` |
| Ingest | Envelope validation, rendering, upsert | `main.py` (`/ingest`) |
| Drop-folder loader | Reads CSV and JSONL files from a folder, posts envelopes | `ingest_drop.py` |
| Chunker | Paragraph chunker, pure function | `chunker.py` |
| Embedder | Local embeddings through Ollama | `embeddings.py` |
| Search | Hybrid text + vector search | `main.py` (`/search/hybrid`) |
| Ask | Retrieve, plan, query, answer | `main.py` (`/ask`) |
| LLM calls | Plain HTTPS to the configured provider | `ai_processor.py` |
| Web page | Question box, answer, citations | `capsule.html` |
| MCP server (*optional*) | Lets an MCP client search Capsule | `capsule_mcp.py` |

## 4. The store

PostgreSQL 16 or newer with the pgvector extension, version 0.5 or newer (HNSW indexes). The reference implementation is developed on PostgreSQL 17.

### 4.1 `entries`: one row per source record

| Column | Type | Meaning |
|---|---|---|
| `id` | integer | Capsule's own id. Citations carry it so a client can open the record. |
| `source` | text | Connector instance name, e.g. `rental_erp`. |
| `record_table` | text | Entity type in the source, e.g. `rentals`. |
| `record_key` | text | The record's primary key in the source. Stable across loads. |
| `fields` | jsonb | The record's fields as the connector sent them. |
| `title` | text | Rendered from the schema's `title_field`, else `table key`. |
| `raw_content` | text | The rendered text (§5.3). It is what gets embedded and what the model sees. |
| `source_url` | text | Optional deep link to the record in the source application. |
| `content_hash` | text | Hash of fields, text, url and deleted flag. Unchanged records are skipped. |
| `source_updated_at` | timestamptz | The source's own modification time, when the connector sends one. |
| `deleted_at` | timestamptz | Set when the source reports the record deleted. Excluded from search and views. |
| `created_at`, `updated_at` | timestamptz | When Capsule first stored and last changed the row. |

`(source, record_table, record_key)` is unique. The table also carries a few columns from Capsule's earlier life as a conversation archive (`summary`, `project_id`, `was_split`, `split_lineage`). Records leave them empty.

Full-text indexes: GIN on `to_tsvector('english', title)` and on `to_tsvector('english', raw_content)`.

### 4.2 `embeddings`: one row per chunk

| Column | Type | Meaning |
|---|---|---|
| `id` | integer | |
| `entry_id` | integer | The record this chunk belongs to. |
| `chunk_text` | text | The chunk as embedded. |
| `vector` | vector(768) | Embedding from `nomic-embed-text`. |

Index: HNSW on `vector` with `vector_cosine_ops`, pgvector defaults (`m=16`, `ef_construction=64`).

The vector width is fixed at 768 in the schema. A different embedding model with a different width needs a migration and a full re-embed.

### 4.3 `record_schemas`: what the fields mean

One row per `(source, record_table)`. Connectors send it (see INTERFACE §3). A table with no schema still loads and is searchable, but it gets no SQL view until a schema is declared. The drop-folder loader always declares one.

| Column | Type | Meaning |
|---|---|---|
| `source`, `record_table` | text | |
| `key_field` | text | Which field holds the key. Documentation for people and the model. |
| `title_field` | text | Field used as the record's title. |
| `description` | text | One or two sentences on what a row is. Sent to the model. |
| `mode` | text | `upsert` (default) or `snapshot` (§5.5). |
| `fields` | jsonb | `{field: {"type": ..., "description": ...}}` |
| `updated_at` | timestamptz | |

Types: `text`, `integer`, `numeric`, `boolean`, `date`, `timestamp`.

### 4.4 The `data` schema: typed views for SQL

For every table with a schema, Capsule generates a view `data.<source>__<table>`, for example `data.rental_erp__rentals`. Each view has:

- `_entry_id`, `_key`, `_url`, `_updated_at`
- one column per declared field, cast to its declared type

Views exclude deleted records. Casts use safe-cast functions that return NULL for a value that doesn't parse, so one bad date can't break every query against the view. Ingest reports those values (INTERFACE §2.4). Views are regenerated whenever a schema changes.

The role `capsule_reader` can use schema `data` and select from its views, and nothing else. The `/ask` query step (§7.3) runs as this role.

### 4.5 Migrations

The schema is created and changed only by Alembic: `alembic upgrade head` on an empty database builds everything, indexes and role included. If the database user can't create roles, an administrator creates `capsule_reader` and grants it to that user first.

## 5. Ingest

### 5.1 Entry points

- `POST /ingest` with a batch of envelopes (INTERFACE §4.1).
- `POST /ingest/schema` with a schema message (INTERFACE §3).
- `POST /ingest/snapshot` to close a snapshot load (INTERFACE §5.3).
- The drop-folder loader, `ingest_drop.py`, which turns files into the same two calls (INTERFACE §4.2).

### 5.2 Per record

1. **Validate** the envelope. Invalid records are reported by index and skipped; the rest of the batch loads.
2. **Stale check.** If the envelope has `updated_at` and it is older than the stored `source_updated_at`, skip it as stale.
3. **Hash** fields, text, url and the deleted flag. If the hash matches the stored `content_hash`, the record is unchanged: stop.
4. **Delete.** If `deleted` is true, set `deleted_at` and remove the record's embeddings. The row stays. Stop.
5. **Render** the text (§5.3).
6. **Upsert** the `entries` row on `(source, record_table, record_key)`.
7. **Queue** the record for embedding.

The HTTP response returns after step 6, with counts. Embedding happens in the background.

### 5.3 Rendering

A record's text is what search matches against and what the model reads. It is plain and predictable:

```
rentals R-2291: 19 ft scissor lift
rental_id: R-2291
customer_id: C-101
customer_name: Cedar Flats Paving
item_id: A-104
item_name: 19 ft scissor lift
category: lifts
out_date: 2026-05-04
due_date: 2026-05-18
returned_date: 2026-05-18
days: 14
amount: 1260.00
damage_notes: Lift cylinder weeping hydraulic fluid. Platform would not hold height.
last_modified: 2026-05-18T17:15:00
```

This is rental R-2291 from the sample data in `examples/`. The first line is `table key: title`. Then one `field: value` line per field, in schema order, skipping empty values. If the connector sent a `text` value, it is appended after a blank line. Long text fields stay intact; the chunker splits them.

Connectors control most of the quality here. A rental record that says `customer_id: 881` is hard to find by asking about a customer. One that also carries `customer_name: Cedar Flats Paving` is easy. INTERFACE §6 covers this.

### 5.4 Chunking and embedding

The chunker (`chunker.py`) splits text on blank lines and packs paragraphs into chunks of about 1,500 characters. Each new chunk starts with roughly the last 150 characters of the one before it, and a final chunk under 200 characters is merged into the previous one. It never truncates; every paragraph lands in at least one chunk. Most records produce one chunk.

Each chunk is embedded with `nomic-embed-text` through a local Ollama server (`OLLAMA_URL`, `OLLAMA_EMBED_MODEL`). A vector that comes back with the wrong width is rejected. A chunk that fails to embed is logged and skipped. Its record stays searchable by text, and `repair_embeddings.py` fills the gaps later.

Embedding is the slow step of ingest, especially on CPU-only machines. Load a large source once, after hours; incremental loads after that are small.

### 5.5 Upsert and snapshot modes

- **upsert** (default): each load adds and updates records. Deletes arrive only as envelopes with `deleted: true`.
- **snapshot**: each load is the complete table. After a snapshot load finishes, records of that table that weren't in it are marked deleted. Use it for sources with no modification timestamp and no delete feed. Snapshot loads are declared per table in the schema, and the loader marks the end of a snapshot explicitly (INTERFACE §5).

## 6. Search

`GET /search/hybrid?q=...&limit=30` combines two scores per record:

- **Text score**, from Postgres full-text search (`websearch_to_tsquery`) over title and body: 1.0 when the title matches, 0.75 when only the body does.
- **Semantic score**, `1 - cosine distance` between the query's embedding and the record's best-matching chunk. One chunk per record counts, so a long record can't fill the results with its own chunks.

The final score is `0.4 × text + 0.6 × semantic`. Results under 0.25 are dropped. Deleted records never appear. Each result carries the record's id, source, table, key, title, url, score and the best-matching chunk.

The query embedding uses the same local model as ingest, so search sends nothing off the machine.

## 7. Ask: the query layer

`POST /ask` turns a question into an answer with citations. It makes two calls to the configured LLM and runs at most one read-only SQL query, plus one retry of each if the query fails (§7.3).

```json
POST /ask
{ "question": "Which lifts came back with hydraulic problems this year, and who had them out?" }
```

### 7.1 Retrieve

Run hybrid search (§6) for the question and keep the top `k` records (default 12). Each record is labeled `R1` to `Rk`. Its rendered text is cut to 2,000 characters.

### 7.2 Plan

First LLM call. The model receives:

- today's date and the server's time zone, so "this year" and "last 60 days" resolve
- the catalog: every view in `data`, its description, columns, column types and descriptions, and row counts
- the retrieved records `R1` to `Rk`
- the question

It must return JSON: `{"sql": "SELECT ..."}` when the question needs counting, summing, filtering by date or value, listing every match, or finding what's missing ("customers with no rental in a year"); `{"sql": null}` when the retrieved records are enough. Search finds records similar to the question. It can't find a record that isn't there, so questions about gaps always go to SQL. The prompt requires every query to select the `_key` column of the rows it lists, so those rows can be cited.

### 7.3 Query

If the plan has SQL, Capsule checks it and runs it:

- one statement, starting with `SELECT` or `WITH`; anything with a second statement is rejected
- runs in a `READ ONLY` transaction as `capsule_reader`, which can only read `data` views
- `statement_timeout` of 5 seconds
- at most 200 rows come back; the answer step is told when the cap was hit

If the SQL fails, the error goes back to the model once for a corrected query. A second failure falls back to answering from the retrieved records, and the response says the query failed.

Result rows are labeled `Q1` to `Qn` and keep their `_key`.

### 7.4 Answer

Second LLM call. The model receives the question, today's date, the records `R1`–`Rk`, the query and its rows `Q1`–`Qn`. The rules it is given:

- Answer only from the records and rows provided.
- Cite each claim with the labels it came from, like `[R3]` or `[Q2]`.
- If the material doesn't answer the question, say so and say what's missing. Don't guess.
- Give numbers exactly as the rows show them.

### 7.5 Response

```json
{
  "answer": "Three lifts came back with hydraulic problems in 2026. The 19 ft scissor lift (A-104) was returned by Cedar Flats Paving on May 18 with a weeping lift cylinder [R1]. ...",
  "citations": [
    {"label": "R1", "entry_id": 412, "source": "sample_rental", "table": "rentals",
     "key": "R-2291", "title": "19 ft scissor lift", "url": null}
  ],
  "sql": "SELECT _key, item_name, customer_name, returned_date FROM data.sample_rental__rentals WHERE category = 'lifts' AND damage_notes ILIKE '%hydraul%' AND returned_date >= '2026-01-01'",
  "rows_returned": 2,
  "row_cap_hit": false,
  "grounded": true,
  "model": "claude / <model id>",
  "sent": {"records": 12, "rows": 2, "characters": 9318}
}
```

- The query matched two rentals by the word "hydraulic". The other lifts in the answer came from search by meaning (a pump fitting leaking oil, a boom drifting down), which is why the two paths run together.
- `citations` lists only labels that appear in the answer and that were actually supplied. A label the model made up is removed.
- `grounded` is false when the answer cites nothing. Clients should show such an answer as unsupported.
- `sent` counts what went to the LLM provider for this question (§9).

### 7.6 What the query layer is not

It doesn't keep conversation state; each question stands alone. It doesn't call tools beyond the one query. It doesn't learn from past questions. Those are deliberate: each question is cheap to reason about, and every answer can be traced to what was sent.

## 8. LLM providers

`AI_PROVIDER` selects one of three, each called over plain HTTPS with no vendor SDK:

| `AI_PROVIDER` | Endpoint | Key and model |
|---|---|---|
| `claude` | Anthropic Messages API | `ANTHROPIC_API_KEY`, `ANTHROPIC_MODEL` |
| `openai` | OpenAI Chat Completions API, or any compatible endpoint | `OPENAI_API_KEY`, `OPENAI_MODEL`, `OPENAI_BASE_URL` (default `https://api.openai.com/v1`) |
| `ollama` | Local Ollama `/api/generate` | `OLLAMA_URL`, `OLLAMA_MODEL` |

Each provider has a default model in the code; the `*_MODEL` variable overrides it. You hold the key, and the provider bills you directly.

`OPENAI_BASE_URL` points the `openai` provider at any server that speaks the same API: RouterLM, or a local engine such as llama.cpp, LM Studio or vLLM.

Adding a provider means one function that takes a system prompt and a user prompt and returns text. Nothing else in the system knows which provider answered.

Model choice matters most in the plan step. Writing correct SQL against an unfamiliar schema is harder than summarizing records, and small local models get it wrong more often. Test your own seed questions on the model you plan to use.

## 9. What leaves the machine

| When | What goes out | Where |
|---|---|---|
| Ingest | Nothing. Embeddings are computed locally. | — |
| Search | Nothing. | — |
| `/ask`, cloud provider | The question; the catalog (view, column and field names, descriptions, row counts); up to `k` retrieved records, 2,000 characters each; up to 200 query rows | Your LLM provider, under your key and its data terms |
| `/ask`, `AI_PROVIDER=ollama` | Nothing. | — |
| MCP client (*optional*) | Search results the client requests | The model behind that client, under its terms |

Fully local operation needs no separate build: set `AI_PROVIDER=ollama` and pick a local model. [RouterLM](https://github.com/rene-kropf/routerlm), a separate MIT spec, puts one service in front of every model, local and cloud, so work keeps going when a provider stops and private requests stay in the building. Point `OPENAI_BASE_URL` at it.

If your records carry obligations about third parties (patient data, tax return information, privileged material), check whether sending them to a cloud provider is allowed before you set `AI_PROVIDER` to one. The use-case briefs name the ones we know about.

## 10. Interfaces

### 10.1 HTTP API

Every endpoint except `/health` and the static page requires `Authorization: Bearer <CAPSULE_TOKEN>`.

| Method and path | Purpose |
|---|---|
| `GET /health` | Liveness and version. No auth. |
| `GET /` | The web page. Asks for the token once and keeps it in the browser. |
| `POST /ingest` | Load a batch of record envelopes. |
| `POST /ingest/schema` | Declare or update a table's schema. |
| `GET /ingest/cursor?source=&table=` | Newest `updated_at` stored for that table, for incremental loads. |
| `POST /ingest/snapshot` | Close a snapshot load: mark records missing from it deleted. |
| `GET /sources` | Every source and table: record count, deleted count, records waiting for embeddings, last load time, newest `updated_at`. |
| `GET /search/hybrid?q=&limit=` | Hybrid search (§6). |
| `POST /ask` | Question in, answer and citations out (§7). |
| `GET /entries/{id}` | One record: fields, rendered text, url. Citation targets point here. |

### 10.2 Web page

`GET /` serves one HTML file: a question box, the answer with its citations as links, and a panel per citation showing the record. It calls the same API as everything else.

### 10.3 MCP server (*optional*)

`capsule_mcp.py` exposes two read-only tools to MCP clients such as Claude Desktop: `search_capsule` (hybrid search) and `get_capsule_entry` (one record). It reads `CAPSULE_URL` and `CAPSULE_TOKEN` from its environment. The client's own model does the answering. Its answers don't go through the grounding checks of §7, and the data it reads falls under that client's terms.

## 11. Security

- **One token.** `CAPSULE_TOKEN` (a long random string in `.env`) guards the API. The server refuses to start without one.
- **Localhost by default.** Run the server on `127.0.0.1`. For access from other machines, put it behind something that authenticates people first: a VPN, or a proxy that requires a login. Don't expose the port directly.
- **One trust boundary.** Everyone with the token can ask about everything in the store. There are no per-user permissions. If some records must be hidden from some people, run separate instances with separate data.
- **Read-only questions.** Model-written SQL runs as `capsule_reader` in a read-only transaction, one statement, five-second timeout, 200-row cap (§7.3).
- **Keys stay in `.env`.** Provider keys and the token never go in the repository. `.env` is in `.gitignore`.
- **Backups.** `pg_dump` the database nightly. Keep the drop folder's `done/` archive (INTERFACE §4.2) as a second copy of what was loaded.

## 12. Hardware and sizing

Capsule runs on one small machine:

- a desktop mini (around $300 used or entry-level), or
- a budget VPS with 2 vCPUs and 4 GB of RAM (around $5 a month at low-cost hosts).

| Component | Needs |
|---|---|
| PostgreSQL + pgvector | A few hundred MB of RAM at small-business scale |
| Ollama with `nomic-embed-text` | About 0.5 GB of RAM; CPU is fine, slower than Apple Silicon or a GPU |
| Capsule API | Under 200 MB |
| Local chat model (*optional*) | Several GB of RAM for a usable model. On a $5 VPS, use a cloud provider instead. |

Disk: each chunk's vector is 3 KB before indexing. Plan on roughly 0.5 GB per 100,000 single-chunk records, vectors, index and text included. That figure is an estimate; measure your own with `GET /sources` and `pg_database_size`.

Supported platforms: macOS (Apple Silicon and Intel) and Debian/Ubuntu Linux. Windows isn't supported directly. A connector may run on Windows (QuickBooks Desktop, Access) and post to a Capsule server on another machine.

## 13. Freshness

Capsule is as fresh as the last connector run. Connectors run on a schedule chosen per install: every 15 minutes for a busy rental counter, nightly for a law firm's billing. Schedulers already exist: cron, launchd, systemd timers, FileMaker Server schedules, Windows Task Scheduler. Capsule doesn't add another.

`GET /sources` shows, per table, when the last load arrived and the newest source timestamp. If a table should load nightly and hasn't loaded in two days, something upstream stopped. Checking this endpoint is the whole monitoring story.

## 14. Limits

- **Read-only.** It answers questions. It doesn't change records, send messages or take actions.
- **Not real-time.** See §13.
- **As good as the schema descriptions.** The model writes SQL from the catalog. A column called `amt2` with no description will be misread. Field descriptions are the cheapest quality improvement there is.
- **The model can still misread.** Citations exist so a person can check. Treat an answer as a pointer to the records behind it.
- **No per-user permissions** (§11).
- **Single machine.** No clustering, no replicas. At small-business scale one machine is enough.
- **Text and numbers only.** Images, PDFs and attachments aren't loaded unless a connector extracts their text.

## 15. Building a compatible implementation

Another implementation is compatible with Capsule Base when:

1. it accepts the record envelope and schema message of [connectors/INTERFACE.md](connectors/INTERFACE.md) with the same semantics (upsert key, stale check, tombstones, snapshot mode), over `POST /ingest` or a drop folder with the same layout, and
2. its answers cite records by `(source, table, key)` and list only citations that were supplied to the model.

Then any connector written for this repository works with it, and the reverse.
