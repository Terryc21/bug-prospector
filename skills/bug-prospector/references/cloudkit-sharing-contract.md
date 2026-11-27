# CloudKit sharing: server-side contracts the Swift cannot show

*Used by Lens 4 (Data Lifecycle Tracing, the Sync step) whenever the scope contains `CKShare`.
Moved here from radar-suite's roundtrip-radar ("CloudKit Sharing Contract Detection Guide")
when that skill was retired in November 2026.*

## What it finds

CloudKit sharing code where the Swift is locally correct (it compiles, the value is assigned,
nothing throws) but the CloudKit server silently refuses the write or withholds the data.
These contracts hold in every environment, Development included, so Dev testing never
surfaces them.

They fail *successfully*. `try await db.save(share)` returns, the local object shows the new
value, and the owner's UI confirms a change that never landed. Because the source reads
correct, a static audit reports the area clean, and "no finding" looks the same as "verified
safe".

## The contract underneath every pattern

A recipient who joins a share by tapping an invite **link** (while `publicPermission != .none`)
is a CloudKit **public participant** (`role == .publicUser`). Apple's rule, from `CKShare.h`:
to modify the list of participants, a share must have `publicPermission` set to none. Public
and private participants cannot be mixed on one share.

So **link-only invites and per-member roles are mutually exclusive.** Most codebases make that
choice implicitly, in one line, without recording that they made it.

## Patterns

| Pattern | Code signature | What the server does |
|---|---|---|
| Per-participant permission write | `participant.permission = .readWrite` or `.readOnly` on a share whose `publicPermission != .none` | **Silently discards it.** A link-joined member's access is governed by share-level `publicPermission` only. The save succeeds, and the participant's own fresh fetch still returns the old value. Fix: write `publicPermission` (applies to everyone), or switch to `publicPermission = .none` with invited private participants. |
| Nil-only guard on an identity read | Any nil-only guard over identity: `?? "Unknown"`, `if let name = identity.nameComponents?.formatted()`, `guard let ... else` | **The guard never fires.** For a public participant the server withholds identity and `formatted()` returns an **empty string, not nil**. It binds, passes the guard, and reaches the UI ("Export 's Items", "'s Household") or a key (`"name:"`). Fix: test for emptiness, not nil, and fall back name, then email, then a placeholder. Match the nil-only *meaning*, not the `??` spelling: `if let` and `guard let` over the same expression are the same defect. |
| No branch for both missing | A name-then-email fallback with a single terminal placeholder, shown in possessives or list rows | CloudKit can withhold **both** identity and email, so every member collapses to the same placeholder and nobody is distinguishable. An emptiness-aware helper is necessary but not sufficient; the both-missing case needs its own UI. |
| Permission shown from the wrong field | A role chip or edit gate bound to `participant.permission` | **The UI disagrees with reality.** Effective access is the higher of `publicPermission` and any individual grant. Reading the per-participant field shows the dropped write as if it had landed. |

## Two traps that ride along with the fix

1. **Never write `publicPermission = .none` to mean "view only".** It stops the invite link
   working and drops pending participants. `.readOnly` is the view-only value.
2. **A "heal" that forces `.readWrite` back** will silently re-promote a deliberately view-only
   share the next time an invite is handed out. Heal only the unjoinable `.none` case.

## Method

1. Search for the share surface:
   ```
   Grep pattern="CKShare" glob="$SOURCE_GLOB"
   Grep pattern="publicPermission" glob="$SOURCE_GLOB"
   Grep pattern="\.permission\s*=" glob="$SOURCE_GLOB"
   Grep pattern="participant" glob="$SOURCE_GLOB"
   ```
   **Zero hits for `publicPermission` alongside hits for `.permission =` is itself a finding:**
   the code manages per-member roles on a share whose join mode was never considered.
2. For every identity read, check the guard. Search the identity properties, then read each
   hit's guard; `??`, `if let` and `guard let` are all the same defect when they test only
   for nil:
   ```
   Grep pattern="(nameComponents|formatted\(\)|lookupInfo|userIdentity)" glob="$SOURCE_GLOB"
   ```
   Nil-handling alone is not enough; require an emptiness test. This check has the highest
   hit rate of the four.
   Identity used as a **key** is a separate sweep from identity used for **display**. A
   codebase that fixed its display helper may still build keys through an old fallback chain.
3. Trace which field the **UI** reads for role or permission, separately from which field the
   **writer** sets. A mismatch is the wrong-field pattern.
4. **Before diagnosing, look for a sibling manager that already solved it.** These bugs recur
   per manager, and the fix often already exists elsewhere in the same codebase.

## Confidence

Every finding here is `needs-runtime` at best: grep cannot observe the server. Never report one
as verified from static reading. Say what a two-account round trip would show, so the finding
names its own missing evidence.

The signature of a real dropped write: the save succeeds, and the participant's own fresh
server fetch still returns the old permission, immediately, after a force-quit, and after a
long wait (15 minutes rules out propagation lag). To confirm in 30 seconds, log
`participant.role.rawValue` on a freshly fetched share; `.publicUser` settles it.

- **Do not blame the read path first.** Check whether the *owner's own* next fetch shows the
  change; that separates a save-side failure from a read-side one in one step.
- **A blank member name is not a failed join.** Check that the participant's status is
  Accepted and that shared items actually arrive.

## Origin

One app, four sharing defects over six weeks in 2026, one root cause. A legacy share's
"Allow Family Input" was switched on and the recipient stayed view-only (07/05). An owner's
change of a member to viewer never reached the member (08/05); the fix already existed in a
sibling manager. The empty-string defect was logged twice from two devices, with nameless
member rows and a button reading "Export 's Items" (08/05, 08/07). Every identity then
collapsed to "Unknown" because CloudKit withheld both identity and email (08/14). Each shipped
past an audit that reported CloudKit clean, because each one's Swift read correctly.
