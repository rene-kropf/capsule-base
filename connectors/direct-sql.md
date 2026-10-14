<!--
direct-sql.md v0.1 — 2026-10-14
Changelog:
- v0.1 (2026-10-14): First draft, with a stdlib-only SQLite example connector.
-->

# Connector: direct SQL

**Covers:** databases you can query directly with a native driver: PostgreSQL, SQLite, MySQL and MariaDB. Many small-business apps keep their data in one of these, and so do most custom-built systems.

**Status:** a pattern and an example in this chapter. The reference implementation doesn't ship a SQL connector module, because each one is a short script shaped by the source's tables.

## The pattern

For each table people ask about:

1. Write one `SELECT` that returns the fields worth sending, with related names joined in (`customer_name`, not just `customer_id`).
2. Ask Capsule for the table's cursor and add `WHERE updated_at > :cursor` (minus a few minutes of overlap).
3. Turn each row into an envelope and post them in batches of 1,000.

Connect with a **read-only database user**. A connector has no reason to write to the source, and a read-only user makes sure it can't.

## Example: SQLite to Capsule

This script uses only the Python standard library. It sends a `jobs` table from a field-service app's SQLite file. Change the query and the schema for your own tables.

```python
#!/usr/bin/env python3
"""sqlite_jobs.py: send jobs from a SQLite database to Capsule Base."""
import json, os, sqlite3, urllib.parse, urllib.request
from datetime import datetime, timedelta

CAPSULE = os.environ.get("CAPSULE_URL", "http://127.0.0.1:8000")
TOKEN = os.environ["CAPSULE_TOKEN"]
DB = os.environ.get("SOURCE_DB", "fieldservice.sqlite")
SOURCE, TABLE = "fieldservice", "jobs"

SCHEMA = {
    "source": SOURCE, "table": TABLE,
    "description": "One row per job: a visit to a customer site, with estimate and actuals.",
    "key_field": "job_id", "title_field": "summary",
    "fields": {
        "job_id":        {"type": "text"},
        "customer_name": {"type": "text"},
        "summary":       {"type": "text",    "description": "One-line job description"},
        "job_type":      {"type": "text",    "description": "panel upgrade, service call, rough-in, ..."},
        "scheduled":     {"type": "date"},
        "completed":     {"type": "date",    "description": "Empty while the job is open"},
        "estimate":      {"type": "numeric", "description": "Quoted price in USD"},
        "invoiced":      {"type": "numeric", "description": "Final invoiced amount in USD"},
        "labor_hours":   {"type": "numeric"},
        "notes":         {"type": "text",    "description": "Tech notes from the job"},
    },
}

QUERY = """
SELECT j.id AS job_id, c.name AS customer_name, j.summary, j.job_type,
       j.scheduled, j.completed, j.estimate, j.invoiced, j.labor_hours,
       j.notes, j.updated_at
FROM jobs j JOIN customers c ON c.id = j.customer_id
WHERE j.updated_at > ?
ORDER BY j.updated_at
"""

def call(method, path, body=None):
    req = urllib.request.Request(
        CAPSULE + path, method=method,
        data=json.dumps(body).encode() if body is not None else None,
        headers={"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"})
    with urllib.request.urlopen(req, timeout=120) as r:
        return json.load(r)

def main():
    call("POST", "/ingest/schema", SCHEMA)
    q = urllib.parse.urlencode({"source": SOURCE, "table": TABLE})
    cursor = call("GET", f"/ingest/cursor?{q}")["max_updated_at"]
    since = "1970-01-01T00:00:00"
    if cursor:
        since = (datetime.fromisoformat(cursor) - timedelta(minutes=10)).isoformat()

    con = sqlite3.connect(f"file:{DB}?mode=ro", uri=True)   # read-only
    con.row_factory = sqlite3.Row
    batch, sent = [], 0
    for row in con.execute(QUERY, (since,)):
        fields = dict(row)
        updated = fields.pop("updated_at")
        batch.append({"source": SOURCE, "table": TABLE, "key": str(fields["job_id"]),
                      "updated_at": updated, "fields": fields})
        if len(batch) == 1000:
            sent += report(call("POST", "/ingest", {"records": batch})); batch = []
    if batch:
        sent += report(call("POST", "/ingest", {"records": batch}))
    print(f"{SOURCE}.{TABLE}: {sent} records sent")

def report(result):
    for r in result.get("rejected", []):
        print("rejected:", r)
    for w in result.get("warnings", []):
        print("warning:", w)
    return result["received"]

if __name__ == "__main__":
    main()
```

The comparison `j.updated_at > ?` works because the app stores ISO timestamps as text, which sort correctly. If yours stores Unix seconds or another format, convert the cursor to match.

## PostgreSQL

The same script with `psycopg` (`pip install "psycopg[binary]"`) in place of `sqlite3`:

- Connect as a user with `SELECT` on the tables you send and nothing else.
- Run the query in a read-only transaction: `conn.read_only = True`.
- `updated_at` comes back as a `datetime`; send `updated.isoformat()`.
- `numeric` columns come back as `Decimal`; convert with `float()` or `str()` before `json.dumps`.

Postgres has its own replication and foreign-data wrappers, and it can be tempting to point Capsule straight at the source tables. Don't. Records that go through the envelope get rendered, embedded, tombstoned and described the same way as every other source, and the source database never learns that Capsule exists.

## MySQL and MariaDB

Use `mysql-connector-python` or `PyMySQL`. The pattern is the same. Watch for zero dates (`0000-00-00`), which MySQL allows and Capsule rejects as dates; send them as null.

## Tables without a modification column

Use snapshot mode (INTERFACE §5.3): declare `"mode": "snapshot"`, send the whole table each run, then post the list of keys to `/ingest/snapshot`. Fine up to a few hundred thousand rows; unchanged rows cost nothing on the Capsule side.

## Scheduling

Run the script from cron, launchd or a systemd timer, as in the CSV chapter. Keep the source credentials in the connector's own environment or config file, never in Capsule's `.env`.
