<!--
examples/README.md v0.3 — 2026-10-14
Changelog:
- v0.3 (2026-10-14): Spec-only publish: describes the data; loading instructions come with the reference implementation.
- v0.2 (2026-10-14): SETUP link follows the flat layout.
- v0.1 (2026-10-14): Sample rental data described.
-->

# Examples

`sample_rental/` is a made-up equipment rental company: 12 customers, 14 fleet units and 39 rentals from January to October 2026, with check-in notes. Every name, number and note is invented.

It's the data behind the examples in [ARCHITECTURE.md](../ARCHITECTURE.md) (rental R-2291, the hydraulic-problems question) and a worked case of the drop-folder format in [connectors/csv-drop-folder.md](../connectors/csv-drop-folder.md):

- one CSV per table, named after the table
- a `.schema.json` beside each, with table and field descriptions
- customers and items declared as snapshots (the whole table each time), rentals as upserts keyed by rental number, with `last_modified` as the source timestamp

Instructions for loading it arrive with the reference implementation.
