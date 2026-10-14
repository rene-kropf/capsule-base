<!--
veterinary-clinic.md v0.1 — 2026-10-14
Changelog:
- v0.1 (2026-10-14): First draft.
-->

# Use case: veterinary clinic

A clinic's practice-management software holds years of patients, visits, medical notes, treatments and lab results. Asking it "which dogs over eight came in limping, and what did we find?" means a report nobody has time to build. The clinic isn't going to hire a developer. It wants the answer this week.

## Where the data lives

- **Practice-management software:** clients (owners), patients, appointments, medical notes, treatments and prescriptions, invoices, reminders. On-premises systems often run on SQL Server; cloud systems vary.
- **Lab results:** in the practice software, or in a reference lab's portal with its own exports.

## Connector

- [ODBC](../connectors/odbc.md) or [direct SQL](../connectors/direct-sql.md) for on-premises systems, with the vendor's agreement. Some vendors restrict direct database access in their support terms.
- [REST API](../connectors/rest-api.md) or scheduled exports into the [CSV drop folder](../connectors/csv-drop-folder.md) for cloud systems.

## Load first

| Table | Key | Title | Why |
|---|---|---|---|
| patients | patient id | name + species + breed | Species, breed, birth date, weight, status |
| visits | visit id | patient + date + reason | Reason for visit, doctor, outcome |
| medical notes | note id | patient + date | The findings. The richest text in the system. |
| treatments and prescriptions | line id | patient + item | What was given, when, how much |
| lab results | result id | patient + test | Values, flags, dates |
| reminders | reminder id | patient + vaccine or service | What's due and when |

Send the patient's species, breed and age on every visit and note, so a question about "dogs over eight" can be answered from either.

## Seed questions

**What happened:**

- Which dogs over eight came in for limping in the last two years, and what did we find?
- What did we do the last three times this patient came in for an ear infection, and what worked?
- Which patients had vomiting or diarrhea noted within a week of starting a new NSAID?

**How many, how much, when:**

- Which patients are overdue for rabies or DHPP boosters?
- How many tick-borne disease positives did we have each month for the last three years?
- What's our no-show rate by weekday and appointment type?
- Which clients haven't been in for 18 months?
- What's the average dental cleaning invoice this year against last year?

**Both:**

- Which senior cats had rising kidney values last year, and what do the notes say we recommended?

## Descriptions that matter

- The clinic's own abbreviations in medical notes. List the common ones in the notes field's description.
- Species and breed codes, if the software stores codes rather than names.
- Weight units, and which date a lab result carries (drawn or reported).

## Watch for

- **Confidentiality.** Animal records aren't covered by HIPAA, but many US states treat veterinary records as confidential, and client names and contact details are personal data. Check your state's rules before sending records to a cloud model, leave out contact details nobody asks about, or run a local model.
- **It reports what your records say.** Capsule finds patterns in your own case history. It isn't a clinical reference, and it doesn't know anything your records don't.
