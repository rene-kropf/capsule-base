<!--
odbc.md v0.1 — 2026-10-14
Changelog:
- v0.1 (2026-10-14): First draft.
-->

# Connector: ODBC

**Covers:** systems whose data is reachable through an ODBC driver: QuickBooks Desktop, Microsoft Access, Microsoft SQL Server, MySQL. Many on-premises practice-management and line-of-business systems run on SQL Server or Access underneath, even when the vendor doesn't advertise it.

**Status:** a pattern in this chapter. Not shipped as a module in the reference implementation.

## When to use ODBC

Use it when the data sits in a database you can't reach with a native driver, or when the vendor only documents ODBC access. If the same system can schedule a CSV export, compare the two. A scheduled export into the [drop folder](csv-drop-folder.md) has fewer parts. ODBC wins when you need changed-since queries or tables the export screens don't offer.

## The pattern

Same as [direct SQL](direct-sql.md), with `pyodbc` (`pip install pyodbc`) as the driver layer:

```python
import pyodbc
con = pyodbc.connect("DSN=ServiceDB;UID=capsule_ro;PWD=...", readonly=True)
cur = con.cursor()
cur.execute("SELECT ... FROM ... WHERE LastModified > ?", since)
cols = [c[0] for c in cur.description]
for row in cur:
    fields = dict(zip(cols, row))
    ...  # build envelopes as in direct-sql.md
```

Rows come back with Python `datetime`, `Decimal` and `bytes` values. Convert dates with `.isoformat()` and decimals with `float()` or `str()`, and drop binary columns before building envelopes.

## Where the connector runs

ODBC drivers for QuickBooks Desktop and Access are Windows-only. The usual layout:

- Capsule runs on its own machine (a mini or a VPS).
- The connector runs on the Windows machine that has the driver and the data file, on a Task Scheduler schedule.
- It posts envelopes to Capsule over the office network, or over a VPN when Capsule is off-site.

Only HTTP leaves the Windows machine, and only to Capsule.

## QuickBooks Desktop

- QuickBooks Desktop is reached through **QODBC**, a third-party driver. Some QuickBooks Desktop Enterprise editions include a read-only edition of it; otherwise it's a separate purchase. QuickBooks Online has no ODBC; use its REST API (see [rest-api.md](rest-api.md)) or scheduled report exports.
- QODBC works through QuickBooks itself, so it's slow. A full read of a large invoice table can take many minutes. Pull only the columns you need, pull line tables only when people ask about line items, and run loads after hours.
- QuickBooks tables carry `TimeModified`, which works with the cursor (INTERFACE §5.1).
- Good first tables: customers, invoices, invoice lines, bills, items, estimates.
- Match the driver's bitness to your Python (32-bit driver, 32-bit Python), or the DSN won't be found.

## Microsoft Access

- Driver: the Microsoft Access Database Engine (ACE) ODBC driver. Its bitness has to match Python's too.
- Open the file read-only. An Access file that people are using is easy to lock or damage with a careless writer.
- Many Access apps have no modification timestamps. Use snapshot mode (INTERFACE §5.3); Access tables are usually small enough.
- Copying the `.accdb` file and reading the copy is often safer than reading the live file during business hours.

## SQL Server

- Driver: Microsoft ODBC Driver for SQL Server, available for Windows, macOS and Linux. The connector can run on the Capsule machine itself and reach SQL Server over the network.
- Ask the vendor or the IT person for a read-only SQL login. Some vendors forbid direct database access in their support terms; check before connecting to a vendor-managed database.
- Look for `rowversion` columns or `ModifiedDate`-style columns for the cursor. A `rowversion` is a counter rather than a time. Store it in a field and keep your own cursor for it, or fall back to snapshot mode.

## MySQL

If you can install a native driver, use the [direct SQL](direct-sql.md) pattern instead. ODBC adds a layer and buys nothing here.

## Gotchas

- **DSN vs. connection string.** A DSN configured in the ODBC administrator is easiest on Windows. Keep the password out of the script, in the DSN or the connector's own config.
- **Text encodings.** Older Access and SQL Server databases store text in a Windows code page. If names come through garbled, set the driver's encoding options or decode with `cp1252`.
- **Views beat joins in the connector.** If you can create a view at the source (SQL Server, Access queries), put the joins there and select from it. The connector stays a straight copy.
