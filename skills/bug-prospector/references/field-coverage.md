# Field coverage across save and export paths

*Used by Lens 4 (Data Lifecycle Tracing). Moved here from radar-suite's data-model-radar
(Domain 2, "Serialization Coverage") when that skill was retired in November 2026.*

## What it finds

A field that exists on the model but is missing from one of the places the model is written
out and read back: a backup, a CSV export, a CSV import, a sync record, an API mapping. Each
of those paths compiles and works for the fields it knows about. The field it forgot is lost
silently on the round trip, and no single file looks wrong.

The export-then-import gap is where data loss usually hides: a column is exported but never
read back, so a restore or re-import quietly drops it.

## When to run it

Run it for every model that Lens 4 picked as a primary entity and that has **two or more**
write-out paths. A model with one path has nothing to diff against.

## Method

1. Read the model and list every stored property. Skip computed and `@Transient` properties.
2. Find every write-out and read-back path for that model. Check each kind that exists:
   - backup struct or backup encoder
   - CSV (or other file) export
   - CSV (or other file) import
   - sync record mapping (CloudKit `CKRecord`, Firebase, a REST payload)
   - API response mapping, if the model is filled from a server
3. **Read each path's code.** A path is not "covered" until you have read it. "Backup looks
   complete" without opening the backup struct is not a check.
4. Build the coverage table below and diff: a model field absent from a path is a candidate.
5. Classify each candidate before reporting it.

## Coverage table

```
| Field | Model | Backup | Export | Import | Sync | Why excluded / receipt |
|-------|:-----:|:------:|:------:|:------:|:----:|------------------------|
| title | yes | yes | yes | yes | yes | Backup.swift:41, CSVExport.swift:88 ... |
| syncID | yes | yes | no | no | yes | internal metadata |
| room | yes | yes | yes | no | yes | GAP: exported, never imported |
```

- `yes` / `no` only after reading the code; cite file:line in the last column.
- `?` means not yet checked. Do not report the model's coverage while any cell you claimed to
  check still reads `?`.
- A path you did not read gets the whole column marked `not checked`, not a row of guesses.

## Classifying a gap

Not every missing field is a bug. Before reporting one, decide which it is:

| Class | Meaning | Report? | Example |
|---|---|---|---|
| **Gap** | User data lost on the round trip | Yes | `purchasePrice` exported but not imported |
| Format mismatch | The field cannot reasonably live in that format | No, note it | a JSON blob left out of CSV |
| Internal metadata | Sync or system bookkeeping, not user data | No, note it | `cloudSyncID` left out of CSV |
| Relationship data | Child objects with their own serialization | No, if the child is serialized | attachments left out of CSV |
| Scope boundary | The path deliberately covers a subset | No, if the subset is documented | a shared-zone sync that carries only household fields |

Tests that decide it:
- **User data test:** if this field vanished after export and import, would the user notice?
  Yes means a gap.
- **Format test:** could the field be represented in this format at all?
- **Subset test:** does the path document that it covers a subset, and does this exclusion
  match that definition? An undocumented subset is a gap until someone documents it.

## Reporting

Each reported gap is an ordinary bug-prospector finding: file:line of the path that drops the
field, the round trip that loses it, and what the user sees afterwards (for example: "after a
restore, every item's room is blank"). Classified exclusions go in the report's "Already
Guarded" section, so the next run does not re-flag them.
