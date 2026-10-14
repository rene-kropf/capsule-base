<!--
filemaker.md v0.1 — 2026-10-14
Changelog:
- v0.1 (2026-10-14): First draft. Export-script route feeds the CSV drop folder; Data API/OData route points to the REST chapter.
-->

# Connector: FileMaker

**Covers:** FileMaker Pro solutions hosted on FileMaker Server, and FileMaker Cloud.

**Status:** a recipe that uses the built-in [CSV drop folder](csv-drop-folder.md). FileMaker writes CSV files on a server schedule, and Capsule loads them like any other export. No FileMaker-specific code runs in Capsule.

There are two routes:

| Route | Use when |
|---|---|
| **Export scripts** to the drop folder | FileMaker Server on a machine you control. This route has the most production time behind it. |
| **Data API or OData** through a [REST connector](rest-api.md) | FileMaker Cloud, or when Capsule can't get files from the server |

## Route 1: export scripts

### Before you start

Each table you send needs:

- **A stable key.** A UUID primary key field (`PrimaryKey` in tables created since FileMaker 17). Don't use `Get ( RecordID )`; it changes when records are imported or a file is rebuilt.
- **A modification timestamp** (`ModificationTimestamp` in tables created since FileMaker 17). Without one, use snapshot mode for that table.
- **An export layout** holding only the fields to send, with related fields pulled in for names (`Customers::Name` on the rentals export layout). Export Records exports the first related record for related fields, which is right for parent names.

Build and test the scripts in a copy of the file before adding them to a production file.

### The export script

One script per table, run by a FileMaker Server schedule every 15 minutes or nightly:

```
# Capsule — export rentals changed since the last run
Set Error Capture [ On ]
Set Variable [ $since ; Value: Capsule Settings::LastExportRentals - 600 ]   # 10 minutes of overlap
Set Variable [ $now ; Value: Get ( CurrentTimestamp ) ]
Go to Layout [ "Capsule Export Rentals" ]
Enter Find Mode [ Pause: Off ]
Set Field [ Rentals::ModificationTimestamp ; ">" & $since ]
Perform Find [ ]
If [ Get ( FoundCount ) > 0 ]
    Set Variable [ $tmp ; Value: Get ( DocumentsPath ) & "capsule-drop/rental_erp/rentals.tmp" ]
    Export Records [ With dialog: Off ; "$tmp" ; Merge ; Unicode (UTF-8) ]
    Rename Data File [ Source file: "$tmp" ; New name: "rentals.csv" ]
End If
Set Field [ Capsule Settings::LastExportRentals ; $now ]
```

Notes on the steps:

- **Merge format, not CSV.** FileMaker's Merge export is comma-separated with field names in the first row. Its plain comma-separated export has no header row, and the loader needs one.
- **Server paths.** Scripts running on FileMaker Server can only write inside the server's Documents folder or its temporary folder. `Get ( DocumentsPath )` points to the Documents folder.
- **Write, then rename.** Exporting under `.tmp` and renaming means the loader never reads a half-written file. If the previous `rentals.csv` hasn't been loaded yet, the rename fails and the next run's overlap catches up.
- **The cursor lives in FileMaker.** A one-row settings table holds each table's last export time. Capsule's own cursor endpoint (INTERFACE §5.1) works too, through Insert from URL, but then the script has to parse an ISO timestamp and hold the API token.

### Where the drop folder lives

- **Capsule on the FileMaker Server machine:** set `DROP_DIR` to the export folder, e.g. `/Library/FileMaker Server/Data/Documents/capsule-drop` on macOS.
- **Capsule on another machine:** run `ingest_drop.py` on the FileMaker Server machine with `DROP_DIR` set to the export folder and `CAPSULE_URL` pointing at Capsule. It posts envelopes over HTTP; files never leave the server.

### The schema file

FileMaker exports dates and timestamps in the file's own formats, usually `5/18/2026` and `5/18/2026 4:42:00 PM` in US-built files. Declare them in `rentals.schema.json` beside the export:

```json
{
  "description": "One row per rental line: one item out to one customer.",
  "key_field": "PrimaryKey",
  "title_field": "ItemName",
  "updated_field": "ModificationTimestamp",
  "fields": {
    "PrimaryKey":            {"type": "text"},
    "OutDate":               {"type": "date",      "date_format": "%m/%d/%Y"},
    "ReturnedDate":          {"type": "date",      "date_format": "%m/%d/%Y", "description": "Empty while out"},
    "Amount":                {"type": "numeric",   "description": "Billed amount in USD before tax"},
    "ModificationTimestamp": {"type": "timestamp", "date_format": "%m/%d/%Y %I:%M:%S %p"}
  }
}
```

The loader normalizes header names: `OutDate` becomes the field `outdate` and `Customers::Name` becomes `customers_name`. Names in the schema file get the same treatment, so you can write them as they appear in FileMaker. `rentals.schema.inferred.json` shows the final names after the first load.

### Deletes

FileMaker keeps no log of deleted records. Two options:

- **Snapshot** small tables (customers, items, staff): export all records each night with `"mode": "snapshot"`. Records no longer in the export get marked deleted.
- For large tables, have the solution's delete script write the key to a `Deleted Log` table, and export that table's new rows with a `_deleted` column set to `true`.

### What doesn't come across

- Container fields. Export a text field instead (a file name, or text extracted by a script) if people need to find documents.
- Unstored calculations are exported with their current values, which can be slow on large found sets. Prefer stored fields on export layouts.
- Summary fields. Capsule computes totals itself from the rows.

## Route 2: Data API or OData

FileMaker Server and FileMaker Cloud both offer a Data API (JSON over HTTPS), and current versions also offer OData. A connector built on the [REST pattern](rest-api.md):

- logs in with a dedicated account whose privilege set can only view the tables you send
- finds records with `ModificationTimestamp` greater than the cursor, paging through the found set
- builds envelopes from each record's `fieldData` and related portal data
- logs out at the end of the run

Data API and OData traffic counts against FileMaker Server's annual data-transfer allowance. Check how much your license includes before scheduling full loads of large tables.
