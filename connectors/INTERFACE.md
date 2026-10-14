<!--
INTERFACE.md v0.2 — 2026-10-14
Changelog:
- v0.2 (2026-10-14): Spec-only publish wording.
- v0.1 (2026-10-14): First draft of the connector contract: envelope, schema message, HTTP and drop-folder transports, incremental loads, deletes.
-->

# The connector interface

A connector is any program that reads records from a source system and hands them to Capsule as **record envelopes**. It can be written in any language and run on any machine that can reach Capsule over HTTP, or write files into Capsule's drop folder.

The reference implementation includes one connector, the CSV drop folder. The other chapters in this folder show how to write the rest. This page is the contract all of them follow.

| Chapter | Covers |
|---|---|
| [CSV drop folder](csv-drop-folder.md) | Anything that can export a spreadsheet. Built in. |
| [Direct SQL](direct-sql.md) | Postgres, SQLite, MySQL, other databases you can query directly |
| [ODBC](odbc.md) | QuickBooks Desktop, Microsoft Access, SQL Server, MySQL |
| [REST API](rest-api.md) | Salesforce, HubSpot, Zoho, Xero, FreshBooks, practice-management tools |
| [FileMaker](filemaker.md) | FileMaker Pro and FileMaker Server |

## 1. What a connector does

1. Reads records from the source, preferably through a read-only account.
2. Declares each table once with a **schema message** (§3): what a row is, which field is the key, what the fields mean.
3. Turns each record into an **envelope** (§2).
4. Sends envelopes over HTTP (§4.1) or writes them as files into the drop folder (§4.2).
5. On later runs, sends only what changed (§5).

A connector never needs Capsule's database credentials, and Capsule never needs the source's.

## 2. The record envelope

```json
{
  "source": "rental_erp",
  "table": "rentals",
  "key": "R-2291",
  "updated_at": "2026-05-18T16:42:00-05:00",
  "url": "https://erp.example.com/rentals/R-2291",
  "fields": {
    "rental_id": "R-2291",
    "customer_id": "C-118",
    "customer_name": "Cedar Flats Paving",
    "item_id": "A-104",
    "item_name": "19 ft scissor lift",
    "out_date": "2026-05-04",
    "returned_date": "2026-05-18",
    "amount": 1260.00,
    "damage_notes": "Lift cylinder weeping hydraulic fluid. Platform would not hold height."
  }
}
```

### 2.1 Fields of the envelope

| Field | Required | Meaning |
|---|---|---|
| `source` | yes | Name of this connector instance. Same value on every record it sends. |
| `table` | yes | The kind of record: `rentals`, `patients`, `matters`. |
| `key` | yes | The record's primary key in the source, as a string. Must stay the same for the life of the record. |
| `fields` | yes | The record's data, a flat JSON object. May be empty only when `deleted` is true. |
| `updated_at` | no | When the source last changed the record, ISO 8601. Enables the stale check and incremental loads. |
| `url` | no | A link that opens the record in the source application. Shown with citations. |
| `text` | no | Extra text to index with the record: a note body, a document's extracted text. Appended to the rendered fields. |
| `deleted` | no | `true` when the source deleted the record. Capsule marks it deleted and keeps the row. |

`(source, table, key)` identifies a record. Sending the same identity again updates it.

### 2.2 Names

- `source`: lowercase letters, digits and underscores, starting with a letter, at most 24 characters.
- `table`: same characters, at most 36 characters.
- Field names in a schema: same characters, at most 63. Fields with other names are stored and indexed for search but left out of SQL views, with a warning.

These limits keep the SQL view name `data.<source>__<table>` inside Postgres's 63-character identifier limit.

### 2.3 Values

- Strings, numbers, booleans and null. Nested objects and arrays are stored and rendered as compact JSON in the text, and left out of views.
- Dates as `YYYY-MM-DD`. Timestamps as ISO 8601; a timestamp with no zone is read in the Capsule server's time zone.
- Numbers as JSON numbers or plain decimal strings: no currency symbols, no thousands separators. The connector cleans these. (The CSV loader cleans them for you; see its chapter.)
- At most 200 fields and 100,000 characters of `text` per record.

### 2.4 What comes back

`POST /ingest` answers with counts and per-record problems:

```json
{
  "received": 500,
  "inserted": 12,
  "updated": 30,
  "unchanged": 455,
  "deleted": 2,
  "stale": 1,
  "rejected": [{"index": 17, "reason": "missing key"}],
  "warnings": [{"index": 42, "field": "returned_date", "value": "5/18", "reason": "not a date"}]
}
```

- **rejected** records were not stored. Fix and resend them.
- **warnings** records were stored. The named value shows as NULL in SQL views until it's fixed; search still sees the raw value.
- **stale** records had an `updated_at` older than the one already stored and were skipped.
- **unchanged** records matched what was stored and cost nothing. Resending a whole table is safe.

## 3. The schema message

Send one per table before its first records, and again whenever it changes. `POST /ingest/schema`:

```json
{
  "source": "rental_erp",
  "table": "rentals",
  "description": "One row per rental: one item out to one customer for a date range.",
  "key_field": "rental_id",
  "title_field": "item_name",
  "mode": "upsert",
  "fields": {
    "rental_id":     {"type": "text",    "description": "Rental number, e.g. R-2291"},
    "customer_name": {"type": "text"},
    "item_name":     {"type": "text",    "description": "Item as listed in the fleet, e.g. 19 ft scissor lift"},
    "out_date":      {"type": "date",    "description": "Day the item left the yard"},
    "due_date":      {"type": "date",    "description": "Day the item is due back"},
    "returned_date": {"type": "date",    "description": "Day it came back; empty while out"},
    "amount":        {"type": "numeric", "description": "Billed amount in USD before tax"},
    "damage_notes":  {"type": "text",    "description": "Counter staff notes at check-in"}
  }
}
```

| Field | Required | Meaning |
|---|---|---|
| `source`, `table` | yes | As in the envelope. |
| `description` | strongly advised | What one row is. The model reads it. |
| `key_field` | no | Which field holds the key. |
| `title_field` | no | Field used as the record's title in results and citations. |
| `mode` | no | `upsert` (default) or `snapshot` (§5.3). |
| `fields` | yes | Field name to `{type, description}`. Types: `text`, `integer`, `numeric`, `boolean`, `date`, `timestamp`. |

Only declared fields appear in the SQL view. Undeclared fields are still stored and searchable. Unknown keys in the message are ignored, so a loader can keep its own settings in the same file.

The response names the view: `{"status": "ok", "view": "data.rental_erp__rentals", "columns": 8}`.

**Descriptions do most of the work.** The model writes SQL from table and field descriptions. `amount` with "billed amount in USD before tax" gets summed correctly. `amt2` with no description gets guessed at. Give units, code meanings (`status: O open, C closed, V void`) and anything a new employee would have to ask about.

## 4. Transports

### 4.1 HTTP

```bash
curl -s http://capsule.local:8000/ingest \
  -H "Authorization: Bearer $CAPSULE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"records": [ { "source": "rental_erp", "table": "rentals", "key": "R-2291", "fields": { "item_name": "19 ft scissor lift" } } ]}'
```

- At most 1,000 records and 10 MB per request. Send larger loads in batches.
- A batch is not all-or-nothing. Valid records load; the response lists the rest.
- Requests are idempotent. A retry after a timeout can't create duplicates.

### 4.2 Drop folder

The drop folder (`DROP_DIR`, default `./drop`) is for connectors that can write files but can't make HTTP calls: scheduled exports, FileMaker Server scripts, a person saving a spreadsheet. `ingest_drop.py` reads it and makes the HTTP calls.

```
drop/
  rental_erp/                       ← one folder per source
    rentals.csv                     ← table name = file name
    rentals.schema.json             ← optional schema (§3), stays in place
    rentals.schema.inferred.json    ← written by the loader when it infers types
    customers.jsonl                 ← envelopes, one JSON object per line
    done/
      20261014T070012_rentals.csv   ← loaded files, kept
    failed/
      20261014T070012_items.csv     ← files that failed, kept
      20261014T070012_items.csv.err ← why
```

Rules:

- **Folder name is the source.** File name, without extension, is the table. `.csv` is comma-separated and `.tsv` is tab-separated, header row required. `.jsonl` holds one full envelope per line; its `source` must match the folder.
- **Write atomically.** Write to a name ending in `.tmp`, then rename. The loader also skips any file modified in the last 10 seconds, and files starting with a dot.
- **Schemas first.** Each run sends `*.schema.json` files that changed since the last run, then loads data files. Schema files stay where they are.
- **Nothing is deleted.** A loaded file moves to `done/` with a timestamp prefix. A file that fails moves to `failed/` with an `.err` file beside it. Clearing out `done/` is a person's call.

The CSV specifics (key column, type inference, number cleanup, dates) are in [csv-drop-folder.md](csv-drop-folder.md).

## 5. Incremental loads and deletes

### 5.1 The cursor

`GET /ingest/cursor?source=rental_erp&table=rentals` returns the newest `updated_at` Capsule holds for that table:

```json
{"source": "rental_erp", "table": "rentals", "max_updated_at": "2026-10-14T07:00:12-05:00", "count": 4210}
```

A connector with modification timestamps asks for the cursor, reads rows changed after it, and sends them. Subtract a few minutes from the cursor before querying, to cover clock differences and long transactions. The overlap costs nothing because resent records come back `unchanged`. The connector keeps no state of its own.

### 5.2 Deletes

If the source can say what was deleted (a deleted flag, an audit table, an API "deleted since" call), send those records with `"deleted": true`. Capsule sets `deleted_at`, drops the record from search and views, and keeps the row.

### 5.3 Snapshots

Some sources have no modification timestamps and no record of deletes. For those, send the whole table every run and declare `"mode": "snapshot"` in its schema. After the last batch, close the snapshot:

```json
POST /ingest/snapshot
{"source": "rental_erp", "table": "items", "keys": ["A-101", "A-102", "A-104", "..."]}
```

Records of that table whose keys aren't in the list are marked deleted. If that would mark more than half the table deleted, Capsule refuses and returns the count, since a broken export looks exactly like a mass delete. Send `"allow_large_delete": true` after checking. The drop-folder loader closes snapshots for you.

## 6. Making records read well

Search quality and answer quality depend more on what the connector sends than on anything in Capsule.

- **Bring the names along.** If a rental has `customer_id`, also send `customer_name`. People ask about customers by name, and the model can't join what it can't see.
- **Pick a title a person recognizes.** `title_field` shows up in every citation.
- **Keep free text whole.** Check-in notes, case notes, visit notes and job notes are what search by meaning works best on.
- **Leave out noise.** Internal flags, layout fields, audit columns nobody asks about, binary data.
- **Leave out secrets.** Tax IDs, card numbers, passwords, anything you'd never want quoted in an answer. Whatever reaches Capsule can reach the model at question time.
- **One table per thing people ask about.** Line items can carry their parent's key and its human-readable fields.

## 7. Writing a new connector

1. Pick a source name.
2. List the tables people will ask about. For each, note the key, a title field and a modification timestamp if there is one.
3. Write the schema messages, descriptions included.
4. Read rows through a read-only account. Map each to an envelope. Clean dates and numbers.
5. Choose cursor (§5.1) or snapshot (§5.3) for each table.
6. Send in batches of up to 1,000 and log `rejected` and `warnings`.
7. Keep source credentials in the connector's own config, outside Capsule's `.env`.
8. Schedule it with whatever scheduler the machine already has.
9. Ask your seed questions. When an answer is wrong, the fix is usually a description or a missing name field.

A connector is usually a few hundred lines. With the source's API docs and an AI coding assistant, plan on days per connector. Point it at a scratch database (a second Capsule with its own `DATABASE_URL`) until the load looks right.

If you write one for a system other people use, publish it and open an issue here with the link, so the next person can find it.
