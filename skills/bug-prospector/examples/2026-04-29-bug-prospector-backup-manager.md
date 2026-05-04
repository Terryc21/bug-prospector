# Bug Prospector Report: Backup Manager + Item List View Model

> [!NOTE]
> This is a **sample bug-prospector report** demonstrating what the skill produces against a real project. It uses real Stuffolio source files but the findings shown here are illustrative — the actual project has resolved several of these and the scenarios shown represent earlier-snapshot conditions intentionally surfaced for teaching value. Read this file end-to-end to see the report shape; for live runs, the file lands at `.agents/research/YYYY-MM-DD-bug-prospector-<scope>.md` in your project.

**Date:** 2026-04-29
**Scope:** `Sources/Managers/BackupManager.swift` + `Sources/ViewModels/ItemListViewModel.swift`
**Lenses Applied:** Assumption Audit · Error Path Exerciser · Boundary Conditions · Data Lifecycle Tracing
**Files Analyzed:** 2

## Summary

| Status | Count |
|--------|-------|
| Bugs Found | 4 |
| Fragile Code | 2 |
| OK (Already Guarded) | 3 |
| Needs Review | 1 |

## Issue Rating Table

All BUG and FRAGILE findings rated and sorted by Urgency then ROI:

| # | Finding | Lens | Urgency | Risk: Fix | Risk: No Fix | ROI | Blast Radius | Fix Effort |
|---|---------|------|---------|-----------|-------------|-----|-------------|------------|
| 1 | BackupManager.swift:1414 — `try? rollback()` swallows rollback failure during restore | Error Path | 🔴 Critical | ⚪ Low | 🔴 Critical | 🟠 Excellent | ⚪ 1 file | Trivial |
| 2 | BackupManager.swift:1216-1223 — 8 entity fetches via `try? context.fetch` silently return empty arrays | Error Path | 🟡 High | ⚪ Low | 🟡 High | 🟠 Excellent | ⚪ 1 file | Small |
| 3 | ItemListViewModel.swift:558-580 — count helpers read `cachedFilteredItems` without verifying input matches cached input | Assumption | 🟡 High | ⚪ Low | 🟡 High | 🟠 Excellent | ⚪ 1 file | Trivial |
| 4 | BackupManager.swift:1093 — assumes `archive.url` exists before reading; no guard for partial archive | Boundary | 🟢 Medium | ⚪ Low | 🟢 Medium | 🟢 Good | ⚪ 1 file | Small |
| F1 | BackupManager.swift:1840 — assumes JSON decode succeeds; new field additions to backup schema would silently lose data | Data Lifecycle | 🟢 Medium | 🟢 Med | 🟡 High | 🟢 Good | 🟢 3 files | Medium |
| F2 | ItemListViewModel.swift:445 — filter rebuild relies on @Observable willSet; rapid filter changes can race | Time | 🟢 Medium | 🟢 Med | 🟢 Medium | 🟡 Marginal | ⚪ 1 file | Small |

**Urgency scale:** 🔴 CRITICAL (crash/data loss) · 🟡 HIGH (incorrect behavior) · 🟢 MEDIUM (degraded UX) · ⚪ LOW (cosmetic/minor)

## Detailed Findings

### 1. BackupManager.swift:1414 — `try? rollback()` swallows rollback failure during restore

**Lens:** Error Path Exerciser

**Assumption:** If `restoreBackup()` fails partway through and triggers `rollback()`, the rollback itself will succeed.

**Violation scenario:** A user restores a backup. Halfway through the restore, the SwiftData context save fails (constraint violation, disk full, corrupted data). The catch block calls `rollback()`. The rollback also fails (most commonly: the rollback transaction touches a model the migration didn't anticipate). With `try?`, the second failure is silently discarded.

The user is now in a half-restored state with no error message and no log entry. They reopen the app and see partial data — some old, some new. There is no diagnostic trail to investigate.

**Consequence:** Silent data corruption. The user sees a "restore failed" toast but the actual state of their database is undefined. Sentry never sees the rollback failure because `try?` discarded it.

**Current code:**
```swift
} catch {
    SFLogger.general.error("Restore failed: \(error)")
    try? context.rollback()              // <-- failure here vanishes
    throw RestoreError.failed(underlying: error)
}
```

**Suggested fix:**
```swift
} catch {
    SFLogger.general.error("Restore failed: \(error)")
    do {
        try context.rollback()
    } catch let rollbackError {
        SFLogger.general.error(
            "CRITICAL: rollback after restore failure also failed: \(rollbackError)"
        )
        // Don't rethrow — original error is still the headline cause
    }
    throw RestoreError.failed(underlying: error)
}
```

The rollback failure is logged separately. If a rollback failure ever shows up in Sentry, an engineer immediately knows to investigate database corruption rather than waiting for a user to write in.

---

### 2. BackupManager.swift:1216-1223 — 8 entity fetches via `try? context.fetch` silently return empty arrays

**Lens:** Error Path Exerciser

**Assumption:** Fetching `InsuranceProfile`, `DonationRecord`, `Beneficiary`, `ValueSnapshot`, `ConflictResolution`, `BeneficiaryPreference`, `ActivityEvent`, and `ScoutBookmark` for inclusion in the backup will always succeed.

**Violation scenario:** SwiftData fetch can fail for several reasons under real-world conditions: corrupted store, permission denied (sandbox edge cases), schema mismatch on first launch after a migration. With `try?`, the result is `nil`, falls back to empty array via `??`, and the backup is silently incomplete.

**Consequence:** A backup completes "successfully" but is missing categories of user data. The user trusts the backup. Six months later they restore and discover their insurance records are gone. There is no log trace to point at the failed fetch.

**Current code (8 instances of this shape):**
```swift
let profiles = (try? context.fetch(FetchDescriptor<InsuranceProfile>())) ?? []
let donations = (try? context.fetch(FetchDescriptor<DonationRecord>())) ?? []
let beneficiaries = (try? context.fetch(FetchDescriptor<Beneficiary>())) ?? []
// ...5 more
```

**Suggested fix:** Replace all 8 calls with the project's existing `fetchWithLogging` helper from `ModelContext+Logging.swift`:

```swift
let profiles = context.fetchWithLogging(FetchDescriptor<InsuranceProfile>())
let donations = context.fetchWithLogging(FetchDescriptor<DonationRecord>())
let beneficiaries = context.fetchWithLogging(FetchDescriptor<Beneficiary>())
// ...
```

Behavior is identical on success. On failure, Sentry receives `"ModelContext fetch failed [BackupManager.swift:1216]: <underlying error>"` and an engineer sees which fetch failed for which entity.

---

### 3. ItemListViewModel.swift:558-580 — count helpers read `cachedFilteredItems` without verifying input matches cached input

**Lens:** Assumption Audit

**Assumption:** `itemCount(in:)`, `expiringSoonCount(in:)`, `expiredCount(in:)`, and `activeCount(in:)` all assume that `cachedFilteredItems` was populated by filtering the same `items` argument that's being passed in.

**Violation scenario:** A view calls `viewModel.itemCount(in: itemsForCategoryA)`. Earlier, the cache was populated by `viewModel.cachedFilteredItems = filterItems(itemsForCategoryB)`. The result returned reflects category B's filter, not the actual input. The user sees "12 items" when there are 47.

This isn't theoretical: the cache is populated by other code paths that don't share state with these count helpers. Tests calling these helpers with different inputs already get wrong answers (this is what surfaced the issue).

**Consequence:** Wrong counts shown to users; tests that exercise these helpers fail intermittently when run in parallel.

**Current code:**
```swift
func itemCount(in items: [Item]) -> Int {
    cachedFilteredItems.count                      // <-- ignores `items` parameter
}

func expiringSoonCount(in items: [Item]) -> Int {
    cachedFilteredItems.count(where: { $0.isDueSoon && !$0.isExpired })
}
```

**Suggested fix:**
```swift
func itemCount(in items: [Item]) -> Int {
    filterItems(items).count                       // <-- always reflects the input
}

func expiringSoonCount(in items: [Item]) -> Int {
    filterItems(items).count(where: { $0.isDueSoon && !$0.isExpired })
}
```

Trade-off: minor performance hit on hot paths since the filter rebuilds. If profiling shows this matters, introduce an input-keyed cache instead of a singleton cache.

---

### 4. BackupManager.swift:1093 — assumes `archive.url` exists before reading; no guard for partial archive

**Lens:** Boundary Conditions

**Assumption:** When the user selects an archive file to restore, the file's URL is reachable and the archive is complete (not partially downloaded from iCloud).

**Violation scenario:** User selects an archive that's still downloading from iCloud. `archive.url` resolves to a placeholder file. `Data(contentsOf: archive.url)` either reads zero bytes or partial bytes. The decode step downstream throws a generic `DecodingError`, not a "file isn't ready yet" error. The user retries five times before realizing they need to wait for iCloud to finish.

**Consequence:** Confusing failure mode; user thinks the backup is corrupted when it's actually still downloading.

**Suggested fix:** Add a `FileManager.default.coordinatedRead(...)` step or check `URLResourceKey.ubiquitousItemDownloadingStatus` before reading. If the status is `.notDownloaded`, surface "your archive is still downloading from iCloud — try again in a minute" instead of a generic decode failure.

---

## Fragile Code (Works Now, May Break Later)

### F1. BackupManager.swift:1840 — assumes JSON decode succeeds; new field additions to backup schema would silently lose data

**Lens:** Data Lifecycle Tracing

**Current behavior:** The backup format encodes each entity as Codable JSON. Decode is forgiving: missing fields use defaults, extra fields are ignored. Today this is fine.

**Breaking scenario:** Future schema changes (the V3 migration that's currently deferred per UNFORGET row P1) add new fields. A user takes a backup on the new schema, then restores it on a device still on the old schema. The decode succeeds (extra fields ignored), but the user's new-schema-only data is silently dropped.

**Recommendation:** Add a schema version field to the backup manifest. On restore, if the manifest version is newer than the runtime, refuse the restore with a clear "this backup was created on a newer version of the app" error. Track in UNFORGET as a follow-up to P1.

---

### F2. ItemListViewModel.swift:445 — filter rebuild relies on @Observable willSet; rapid filter changes can race

**Lens:** Time-Dependent Bugs

**Current behavior:** When the user changes a filter, `@Observable` triggers `willSet`, which kicks off `cachedFilteredItems = filterItems(allItems)`. Single change: works fine.

**Breaking scenario:** User taps three filter chips in quick succession (under 500ms). Three filter rebuilds queue. The order they complete in is not the order they were initiated. Most of the time the last rebuild wins; sometimes a stale earlier rebuild lands last and the displayed count reflects an older filter set.

**Recommendation:** Move filter rebuilds into an explicit Task, cancel prior tasks on new filter changes. Or add a `currentFilterEpoch: Int` that's incremented on every filter change and checked before assigning the result.

This rarely surfaces in development because rapid filter-tapping is uncommon in test scenarios. It could surface in QA stress testing or in a power user's normal flow on a slow device.

---

## Already Guarded (Reference)

These were flagged by the search accelerators but verified to be correctly handled:

- **BackupManager.swift:1450** — `try? FileManager.default.removeItem(at: tempURL)` for cleanup of the temp working directory. Verified OK: cleanup failure is non-blocking; the system clears tmp on next reboot. Comment in code explains intent.
- **ItemListViewModel.swift:200** — `items.first!` force unwrap. Verified OK: caller is inside a guard `if !items.isEmpty` block 5 lines above. Force unwrap is intentional and safe.
- **BackupManager.swift:1652** — `dateFormatter.date(from: dateString)!` returning Date. Verified OK: `dateString` is never user input; it's always a formatter-roundtrip from a known schema.

## Needs Human Review

- **BackupManager.swift:1102** — `let archiveSize = try? FileManager.default.attributesOfItem(atPath: ...)[.size] as? Int64 ?? 0`. Returning 0 on error is OK if `archiveSize` is purely informational (shown to user as "archive size: 0 bytes" worst case). If any downstream code branches on `archiveSize > threshold`, the silent zero could route restore through the wrong code path. Domain knowledge needed.

---

*Report generated 2026-04-29 by bug-prospector v1.0.0. Full skill documentation: [SKILL.md](../SKILL.md).*
