<!--
rest-api.md v0.1 — 2026-10-14
Changelog:
- v0.1 (2026-10-14): First draft.
-->

# Connector: REST API

**Covers:** cloud systems with a web API: Salesforce, HubSpot, Zoho, Xero, FreshBooks, QuickBooks Online, and most cloud practice-management, field-service and property-management tools.

**Status:** a pattern in this chapter. Not shipped as a module in the reference implementation. Each API differs in auth, paging and naming, and each connector stays a short script you own.

## Check for an export first

Most of these systems can also schedule a CSV report by email or to cloud storage. If a nightly export covers the questions people ask, land it in the [drop folder](csv-drop-folder.md) and skip the API. The API path is worth it when you need records changed in the last 15 minutes, objects the export screens don't offer, or deletes.

## The pattern

1. **Authenticate.** Usually OAuth 2.0 with a refresh token, sometimes an API key. Register your own app with the vendor, so the credentials belong to the business, and keep tokens in the connector's own config file.
2. **Get the cursor** from Capsule for each object (INTERFACE §5.1).
3. **List records modified since the cursor.** Most of these APIs have a modified-since filter or header; use it with a few minutes of overlap.
4. **Page through results.** Follow the API's paging (next-page token, offset or cursor) until it runs out.
5. **Flatten** each record into `fields`: nested objects become `parent_child` fields, lists become counts or joined text, and related names come along (`account_name`, not just `account_id`).
6. **Post envelopes** in batches of up to 1,000, with `url` set to the record's page in the vendor's app so citations open it.
7. **Handle deletes** if the API reports them (a "deleted since" endpoint or a deleted flag). If it doesn't, run a periodic snapshot of keys (INTERFACE §5.3).

## Rate limits and failures

- Respect the vendor's limits. Read the rate-limit headers, back off on HTTP 429, and spread big first loads over several runs if needed.
- Make every run restartable. Because Capsule's cursor is the newest record stored, a run that dies halfway resumes from where it stopped on the next run.
- Log the API's error body. Most failures are expired tokens or a permission the integration user lacks.

## Notes per system

These are starting points. APIs change, so check each vendor's current documentation.

- **Salesforce.** SOQL through the REST API: `SELECT ... FROM Opportunity WHERE SystemModstamp > <cursor>`. Use a dedicated integration user with read-only permissions. Record links follow `https://<your-domain>.lightning.force.com/lightning/r/<Object>/<Id>/view`. Deleted records can be fetched with the `getDeleted` call or queried with `queryAll`.
- **HubSpot.** The CRM search endpoints filter on each object's last-modified property. Associations (contact ↔ company ↔ deal) are separate calls; bring the associated names into the record.
- **Zoho CRM.** Record listing accepts a modified-since header. Field API names differ from screen labels; map them in the schema descriptions.
- **Xero.** The Accounting API accepts an `If-Modified-Since` header on most endpoints. Invoices carry line items inline; send lines as their own table if people ask about them.
- **FreshBooks, QuickBooks Online.** Both offer OAuth APIs for invoices, clients, expenses and time entries, with update-time filtering on most lists.
- **Practice-management tools** (veterinary, legal, property, tax). API access varies widely. Some vendors offer a full API, some a partner program, some only exports. Ask the vendor what's available before planning a connector, and read their API terms.

## Shape of a connector

```python
def run():
    for obj in ["accounts", "opportunities", "activities"]:
        send_schema(obj)
        since = cursor(obj) - overlap
        for page in api_pages(obj, modified_since=since):
            send([to_envelope(obj, r) for r in page])
```

The core of a REST connector is `to_envelope`: picking fields, flattening, bringing names along, and writing descriptions that say what each field means in this business. That's also where most of the work goes.

## Keep it replaceable

Vendors change APIs, deprecate versions and change terms. Capsule's core doesn't import any vendor SDK, so an API change breaks one connector, and fixing it means editing one script. Keep it that way: no vendor code in the Capsule server, and no Capsule internals in the connector beyond the envelope.
