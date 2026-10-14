<!--
csv-drop-folder.md v0.2 — 2026-10-14
Changelog:
- v0.2 (2026-10-14): Spec-only publish: SETUP.md references removed.
- v0.1 (2026-10-14): First draft. The built-in connector.
-->

# Connector: CSV drop folder

**Covers:** anything that can export a spreadsheet. Accounting reports, practice-management exports, rental and field-service software, a shared spreadsheet someone keeps by hand. If a system has an Export button, this connector reads its output.

**Status:** the reference implementation's built-in connector (`ingest_drop.py`), and the one to start with when you're unsure.

## How it works

1. Something writes `drop/<source>/<table>.csv`: a scheduled export, a script, a person.
2. `ingest_drop.py` runs on a schedule. For each source folder it sends changed schema files, then turns each data file into record envelopes and posts them to `/ingest`.
3. Loaded files move to `done/`. Failed ones move to `failed/` with an `.err` file saying why.

The folder layout and the general rules (atomic writes, nothing deleted) are in [INTERFACE.md §4.2](INTERFACE.md#42-drop-folder).

```bash
python3 ingest_drop.py            # one pass over DROP_DIR, then exit
python3 ingest_drop.py --dry-run  # parse and report, send nothing
```

## The file

- UTF-8, with or without a byte-order mark. Excel's "CSV UTF-8" option produces this. Plain "CSV" from Excel on Windows is often Windows-1252; the loader rejects bytes that aren't valid UTF-8 and names the line.
- First row is the header. Header names become field names: lowercased, with runs of anything other than letters and digits turned into `_`. `Customer Name` becomes `customer_name`. The `.err` or the inferred schema shows the mapping.
- `.csv` is comma-separated with standard quoting. `.tsv` is tab-separated.
- One file is one table. A file named `Rentals 2026-10.csv` would be table `rentals_2026_10`, which is probably not what you want. Name exports by table.

## Which column is the key

In order:

1. `key_field` in `<table>.schema.json`
2. a column named `_key`
3. a column named `id`
4. the first column, with a warning

The key has to identify the row for good. Row numbers from an export are not keys: they change when the sort changes.

## Types

With no schema file, the loader infers each column's type from its values and writes what it decided to `<table>.schema.inferred.json`:

| Inferred type | When every non-empty value... |
|---|---|
| `integer` | is a whole number |
| `numeric` | is a number, allowing a leading `$`, `€` or `£`, thousands commas, and accounting negatives like `(12.50)` |
| `date` | is `YYYY-MM-DD` |
| `timestamp` | is an ISO 8601 date-time |
| `boolean` | is `true`/`false` or `yes`/`no`, any case |
| `text` | anything else |

For `numeric` and `integer` fields, the loader strips the currency symbol and commas and turns parentheses into a minus sign before sending.

Dates like `3/4/2026` are not inferred, because they read differently in the US and in Europe. Declare the format in the schema file and the loader converts them:

```json
"out_date": {"type": "date", "date_format": "%m/%d/%Y"}
```

The format codes are Python's `strptime` codes.

## The schema file

Copy `<table>.schema.inferred.json` to `<table>.schema.json`, fix any types, and add descriptions. It's the [schema message](INTERFACE.md#3-the-schema-message) plus a few loader settings:

```json
{
  "description": "One row per rental: one item out to one customer for a date range.",
  "key_field": "rental_id",
  "title_field": "item_name",
  "updated_field": "last_modified",
  "url_template": "https://erp.example.com/rentals/{rental_id}",
  "mode": "upsert",
  "fields": {
    "rental_id":     {"type": "text",    "description": "Rental number"},
    "out_date":      {"type": "date",    "description": "Day the item left the yard", "date_format": "%m/%d/%Y"},
    "amount":        {"type": "numeric", "description": "Billed amount in USD before tax"},
    "last_modified": {"type": "timestamp"}
  }
}
```

`source` and `table` come from the folder and file names, so they're left out.

| Loader setting | Effect |
|---|---|
| `updated_field` | Column holding the source's modification time. Sent as the envelope's `updated_at`. |
| `url_template` | Builds each record's `url`. `{field}` is replaced by that field's value. |
| `date_format` | Per field, as above. |
| `mode` | `upsert` (default) or `snapshot`. |

Field names in the schema file are normalized the same way as headers, so `Customer Name` and `customer_name` mean the same field. Fields you list are typed. Columns you don't list are inferred as before. To leave a column out entirely, give it `"type": "skip"`.

## Changes and deletes

- **Exports of changed rows** (preferred when the source can filter by modified date): `upsert` mode. Deletes arrive only if the export has a `_deleted` column; rows with `true` there are marked deleted.
- **Full exports every time** (most report exports): `snapshot` mode. Each file is the whole table. After loading it, the loader closes the snapshot, and rows missing from the file are marked deleted. If a file would mark more than half the table deleted, the loader stops, moves the file to `failed/`, and says so. A half-written or filtered export looks exactly like that.
- Re-dropping the same file is harmless. Unchanged rows cost nothing.

## Scheduling

`ingest_drop.py` does one pass and exits. Run it from the scheduler the machine already has. Every 15 minutes with cron:

```
*/15 * * * * cd /opt/capsule-base && .venv/bin/python ingest_drop.py >> logs/ingest.log 2>&1
```

On macOS, a launchd agent with `StartInterval` does the same job. Schedule the export on the source side a few minutes earlier than the loader.

## When the source is on another machine

The drop folder has to be a local folder on the Capsule machine. Ways to get files there, simplest first:

- The export runs on the Capsule machine (the source app lives there, or exports over the network).
- A shared folder the export writes to and Capsule reads, mounted on the Capsule machine.
- A scheduled `rsync` or `scp` from the source machine.

If the source machine can reach Capsule over HTTP, skip files and run `ingest_drop.py` on the source machine with `CAPSULE_URL` pointing at the server. It posts the same envelopes.

## Gotchas

- **Excel rewrites values.** Opening and resaving a CSV in Excel can turn `00412` into `412` and `1-2` into a date. Feed exports straight to the folder without opening them.
- **Line breaks inside notes** are fine when the field is quoted, which proper CSV writers do. Hand-built CSVs often don't.
- **Totals rows.** Report exports often end with a "Total" row. Its key will be blank or odd; the loader rejects rows with a blank key and reports them.
- **Several tables in one report.** Export them separately or split the file before dropping it.
