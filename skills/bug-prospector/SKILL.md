---
name: bug-prospector
description: 'Mine for hidden bugs that pattern-based auditors miss. 7 analysis lenses: assumptions, state machines, boundary conditions, data lifecycle, error paths, time-dependent behavior, and platform divergence. Triggers: "prospect for bugs", "find hidden bugs", "assumption audit", "what could go wrong", "bug prospector".'
version: 1.3.0
author: Terry Nyberg, Coffee & Code LLC
license: Apache-2.0
allowed-tools: [Grep, Glob, Read, Write, Bash, AskUserQuestion, Agent]
metadata:
  tier: analysis
  category: debugging
---

# Bug Prospector

> **Quick Ref:** Find bugs that pattern-based scanners miss by analyzing *intent*, not syntax. Output: `.agents/research/YYYY-MM-DD-bug-prospector-*.md`

**YOU MUST EXECUTE THIS WORKFLOW. Do not just describe it.**

**Required output:** Every BUG finding MUST include Urgency, Risk, ROI, and Blast Radius ratings using the Issue Rating Table format. Do not omit these ratings.

**Philosophy:** Auditing tools find what's *syntactically wrong*. Bug Prospector finds what's *semantically fragile* — the gap between "the code compiles" and "the code handles every real-world scenario."

---

## Pre-flight: Git Safety Check

This check decides whether the user has a revert point before the skill edits anything. **An empty result is only trustworthy if the command actually ran**, so establish that first — run all three and read them together:

```bash
git rev-parse --show-toplevel     # positive control: proves git works AND names the real root
git status --porcelain | wc -l    # 0 = clean, N = N changed files
git status --short                # the listing itself
```

🛑 **Empty output alone is NOT proof of a clean tree.** `git status --short` prints nothing and exits **0** in cases that are not "clean":

- **The repo root is a parent directory.** Run from a subdirectory that has no `.git` of its own, git resolves upward — `--show-toplevel` returning a path ABOVE the directory you targeted means your scope and the repo root differ. Scope paths accordingly.
- **A sandbox or permission layer blocked the read** (macOS TCC on an external volume is the common case). The command appears to succeed and reports nothing.

**Read the three results as a set:**

| `--show-toplevel` | `wc -l` | Meaning |
|---|---|---|
| a path | `0` | Genuinely clean — proceed |
| a path | `N > 0` | N uncommitted files — ask below |
| fails / non-zero exit | anything | **Git is not usable here. Stop and say so** — do not report "clean" |

**If the positive control fails, do NOT proceed as though the tree were clean.** Report that the git state could not be established, name what failed, and let the user decide. A skill that edits files after a failed safety check has removed the user's revert point without telling them.

If uncommitted changes exist:

```
AskUserQuestion with questions:
[
  {
    "question": "You have uncommitted changes. Commit before proceeding?",
    "header": "Git",
    "options": [
      {"label": "Commit first (Recommended)", "description": "Save current work so you can revert if this skill modifies files"},
      {"label": "Continue without committing", "description": "Proceed — I accept the risk"}
    ],
    "multiSelect": false
  }
]
```

If "Commit first": Ask for a commit message, stage changed files, and commit. Then proceed.

### Freshness

Base all findings on current source code only. **Never use a prior report as a source of findings** — not files in `.agents/`, not `scratch/`, not cached findings from auto-memory or previous sessions. Every finding must come from scanning the actual codebase as it exists now. A finding carried forward from an earlier run has not been re-verified against code that may since have changed.

**One exception, and it is not a findings source:** before writing the report, check whether the target filename already exists in `.agents/research/`. The name carries only a date and scope, so a second run on the same scope the same day would otherwise overwrite the first without a word. If it exists, say so and write alongside it with a numeric suffix rather than replacing it.

---

## Step 0: Confirm platform context

Before any lens runs, establish whether the codebase targets a single platform or multiple platforms. This is non-negotiable: every lens needs to know which `#if os(...)` blocks count as "live code" and which to skip.

```
AskUserQuestion with questions:
[
  {
    "question": "What platforms does this project ship to?",
    "header": "Platform",
    "options": [
      {"label": "iOS only", "description": "Skip findings inside #if os(macOS) / #if os(watchOS) / #if os(tvOS) blocks"},
      {"label": "macOS only", "description": "Skip findings inside #if os(iOS) / #if os(watchOS) / #if os(tvOS) blocks"},
      {"label": "Universal (iOS + macOS)", "description": "All non-iOS-only and non-macOS-only blocks count; honor #if conditionals for cross-platform consistency"},
      {"label": "Other / multi-platform", "description": "I'll describe the platform set"}
    ],
    "multiSelect": false
  }
]
```

Record the answer. Every subsequent lens MUST honor it during classification (see Step 3, rule 7).

### Distribution channel (drives the 1-Star Risk column)

The platform answer also determines whether the report carries a **1-Star Risk** column (Step 4.2). Do NOT ask a second question when the platform answer already settles it:

| Step 0 answer | App Store distribution | Ask again? |
|---|---|---|
| iOS only | Yes — App Store | No |
| macOS only | Yes — Mac App Store | No |
| Universal (iOS + macOS) | Yes — both | No |
| Other / multi-platform | Unknown | **Yes** — ask below |

**Only if the user chose "Other / multi-platform":**

```
AskUserQuestion with questions:
[
  {
    "question": "Does this project ship to a public app store where users leave ratings?",
    "header": "Store",
    "options": [
      {"label": "Yes — public app store", "description": "App Store, Mac App Store, Play Store, or similar. Findings get a 1-Star Risk rating."},
      {"label": "No — no public ratings", "description": "Backend service, internal tool, library, or CLI. Skip the 1-Star Risk column."}
    ],
    "multiSelect": false
  }
]
```

Record the result as **store-distributed: yes/no**. A server, library, or internal tool has no review page, so a review-risk column there is noise — that is the whole reason this is conditional.

---

## Step 0.5: Read the invocation arguments

`/bug-prospector` accepts two optional arguments in either order: a **preset** and a **scope**. Parse them BEFORE Step 1 — each one that is present replaces the corresponding question, and asking anyway ignores what the user already told you.

| Invocation | Lenses | Scope | Prompts skipped |
|---|---|---|---|
| `/bug-prospector` | ask (1.2) | ask (1.1) | none |
| `/bug-prospector quick` | Quick 3 — Assumption, Error Path, Boundary | recently changed files | both |
| `/bug-prospector all` | all 7 | full codebase | both |
| `/bug-prospector <path>` | ask (1.2) | that file or directory | 1.1 only |
| `/bug-prospector quick <path>` | Quick 3 | that path | both |
| `/bug-prospector all <path>` | all 7 | that path | both |

**Parsing rules:**

- The preset is the literal token `quick` or `all`. Anything else is a scope.
- A scope argument is a file path, directory path, or glob. Resolve it relative to the project root. If it matches nothing, say so and fall back to the Step 1.1 question rather than scanning the whole codebase — a typo'd path must never silently widen the run. Whatever the scope resolves to, the empty-scope guard under Step 1.1 runs before any lens does.
- `quick` and `all` are mutually exclusive. If both appear, ask which was meant; do not guess.
- Arguments not recognized as either a preset or a resolvable path: report what was not understood, then fall through to the full interactive flow.

**A preset sets the default scope, not a locked one.** `quick` and `all` carry the scopes in the table above because that is what the documentation promises. An explicit path always wins over a preset's default.

**Announce the resolved run before executing** — one line, so a mis-parse is visible before the expensive part starts:

> Running **Quick 3** (Assumption · Error Path · Boundary) on **files changed in the last 5 commits**.

Step 0 (platform context) still runs regardless of arguments: no invocation form supplies the platform, and every lens depends on it.

---

## Step 0.6: Resolve the source glob

Every scope path and every search accelerator below filters on a file glob. **Resolve it once, here, and substitute it everywhere** — the placeholder `$SOURCE_GLOB` in the rest of this file means the value you resolve now.

**Default: `**/*.swift`.** Swift is where this skill's accelerators and lens examples are deepest. Do not change the default speculatively — change it when the project is measurably not Swift.

**Resolution order, first match wins:**

1. **An explicit path from Step 0.5 names a file.** Take its extension. `/bug-prospector quick src/auth.ts` → `**/*.{ts,tsx}`.
2. **Count what the repo actually contains:**

```bash
git ls-files | sed -n 's/.*\.\([a-zA-Z0-9]*\)$/\1/p' | sort | uniq -c | sort -rn | head -10
```

3. **Map the dominant source extension** to a glob, pairing extensions that travel together:

| Dominant | `$SOURCE_GLOB` |
|---|---|
| `swift` | `**/*.swift` |
| `ts` / `tsx` | `**/*.{ts,tsx}` |
| `js` / `jsx` | `**/*.{js,jsx,mjs}` |
| `py` | `**/*.py` |
| `go` | `**/*.go` |
| `rs` | `**/*.rs` |
| `kt` / `java` | `**/*.{kt,java}` |
| anything else | `**/*.<ext>` for the dominant one |

Ignore `md`, `json`, `yml`, `lock`, and image extensions when picking the dominant one — they are never the source under audit.

🛑 **Git and Grep need DIFFERENT forms of this glob. Resolve both.**

`git` pathspec does not expand brace groups; `ripgrep` (behind the Grep tool) does. The same string in both places silently matches nothing in one of them:

```bash
git diff --name-only HEAD~5 -- "**/*.{ts,tsx}"    # matches 0 files — brace not expanded
git diff --name-only HEAD~5 -- "*.ts" "*.tsx"     # matches correctly — separate pathspecs
```

So carry two values:

- **`$SOURCE_GLOB`** — the brace form, for the `Grep` and `Glob` tools: `**/*.{ts,tsx}`
- **`$SOURCE_PATHSPEC`** — space-separated quoted args, for `git` commands: `"*.ts" "*.tsx"`

For a single-extension language the two are `**/*.swift` and `"*.swift"`. The distinction only bites on multi-extension globs, which is exactly where it would go unnoticed.

**Announce it with the run** (extending the Step 0.5 line):

> Running **Quick 3** on **files changed in the last 5 commits**, matching `**/*.{ts,tsx}`.

### 🛑 On a non-Swift project, the accelerators are advisory

The 34 `Grep` patterns in Step 2 are written in **Swift syntax** — `@Published`, `try?`, `#if os(`, `@Query`. Substituting `$SOURCE_GLOB` makes them run against other languages; it does not translate them. Most will match nothing on TypeScript or Python, and **a zero-match accelerator is not evidence of a clean lens.**

So when `$SOURCE_GLOB` is not `**/*.swift`:

- **Read the files.** Step 2 already requires reading each file completely before analysis; on a non-Swift project that reading IS the analysis, not a supplement to grep.
- **Treat every accelerator miss as uninformative**, never as a finding of "none."
- **Translate the concept, not the token.** The Error Path lens asks what happens when an error path is taken — in TypeScript that is `catch`, `.catch()`, and an unawaited promise, not `try?`. Apply the lens's *goal* (stated at the top of each lens) rather than its pattern list.
- **Say so in the report header:** `Accelerators: Swift-tagged, advisory on this project (source glob `**/*.{ts,tsx}`).`

This is a real limit, not a formality. bug-prospector is deepest on Swift; on other languages the lenses carry the analysis unaided.

---

## Step 1: Choose Scope and Lenses

### 1.1: Determine Scope

**Skip this question entirely if Step 0.5 resolved a scope** (a preset's default or an explicit path). Go straight to the resolution notes below using the scope already determined.

```
AskUserQuestion with questions:
[
  {
    "question": "What should I prospect?",
    "header": "Scope",
    "options": [
      {"label": "Specific file or feature", "description": "I'll tell you which file(s) or feature to analyze"},
      {"label": "Recently changed files", "description": "Analyze files changed in the last 5 commits"},
      {"label": "Full codebase", "description": "Broad scan across all source files (slower, more thorough)"}
    ],
    "multiSelect": false
  }
]
```

**If "Specific file or feature":** Ask which file(s) or feature name.

**If "Recently changed files":**
```bash
git diff --name-only HEAD~5 -- $SOURCE_PATHSPEC | head -30
# $SOURCE_PATHSPEC, not $SOURCE_GLOB — git does not expand braces (Step 0.6)
```

**If "Full codebase":** Use `Glob pattern="$SOURCE_GLOB"` to inventory, then prioritize:
1. ViewModels and Managers (business logic)
2. Data layer (persistence, sync, networking)
3. Views with complex state (forms, multi-step flows)
4. Skip: pure layout views, previews, test files

#### Empty-scope guard (applies to every scope path, including a Step 0.5 explicit path)

🛑 **A scope that resolves to zero files is a STOP, never a clean run.** A report generated from zero files reads exactly like a report from a codebase with no bugs — the user is told they are fine when nothing was examined.

Step 0.6 makes this rarer than it was, but not impossible: a `$SOURCE_PATHSPEC` used where `$SOURCE_GLOB` was meant, a `HEAD~5` window with no source changes in it, or a path that resolves to a directory of assets.

**Count the resolved files before any lens runs.** If the count is 0:

1. **Do not proceed to Step 2.** No lenses, no report, no `.agents/` file.
2. **Say what resolved to nothing**, naming the actual filter used — "`git diff HEAD~5 -- "*.ts" "*.tsx"` matched 0 files" is actionable; "no findings" is not.
3. **Re-check the extension census** from Step 0.6 — if the dominant extension does not match the glob you resolved, the resolution was wrong, not the codebase:

```bash
git ls-files | sed -n 's/.*\.\([a-zA-Z0-9]*\)$/\1/p' | sort | uniq -c | sort -rn | head -10
```

4. **Report what you found and let the user choose:**

> Scope resolved to **0 files** — `git diff HEAD~5 -- "*.ts" "*.tsx"` matched nothing. The last 5 commits touched only `.json` and `.md`. Widen the window, or re-run against a path: `/bug-prospector quick src/`.

A missed accelerator costs speed. An empty scope reported as a clean run costs the user's trust in every future run — that is why this one stops.

### 1.2: Choose Lenses

**Skip this question entirely if Step 0.5 resolved a preset** (`quick` → Assumption + Error Path + Boundary; `all` → all 7).

```
AskUserQuestion with questions:
[
  {
    "question": "Which analysis lenses should I apply?",
    "header": "Lenses",
    "options": [
      {"label": "All 7 lenses (Recommended)", "description": "Full prospecting — assumptions, state machines, boundaries, data lifecycle, error paths, time, platform"},
      {"label": "Quick 3", "description": "Assumptions + Error Paths + Boundaries — highest bug yield"},
      {"label": "Let me pick", "description": "I'll choose specific lenses"}
    ],
    "multiSelect": false
  }
]
```

**If "Let me pick":**

```
AskUserQuestion with questions:
[
  {
    "question": "Select the lenses to apply:",
    "header": "Lenses",
    "options": [
      {"label": "1. Assumption Audit", "description": "List every implicit assumption and what breaks when violated"},
      {"label": "2. State Machine Analysis", "description": "Find unreachable states, simultaneous states, interrupted transitions"},
      {"label": "3. Boundary Conditions", "description": "Zero, one, max values; empty strings; nil chains; off-by-one"},
      {"label": "4. Data Lifecycle Tracing", "description": "Follow data from creation to deletion — find gaps"}
    ],
    "multiSelect": true
  },
  {
    "question": "Additional lenses:",
    "header": "More",
    "options": [
      {"label": "5. Error Path Exerciser", "description": "What happens when every try/catch and optional chain fails?"},
      {"label": "6. Time-Dependent Bugs", "description": "Timezone, locale, rapid actions, slow network, first launch after weeks"},
      {"label": "7. Platform Divergence", "description": "Hardware assumptions, OS version gaps, device-specific behavior"}
    ],
    "multiSelect": true
  }
]
```

---

## Step 2: Execute Selected Lenses

For each file in scope, apply the selected lenses. Read the full file before analysis — never analyze from grep matches alone.

### Lens 1: Assumption Audit

**Goal:** Surface every implicit assumption the code makes, then evaluate what happens when each is violated.

**Process:**
1. Read the file completely
2. For every function/computed property, list assumptions such as:
   - "This array will always have at least one element"
   - "This network call will complete before the view disappears"
   - "The user will have iCloud enabled"
   - "This date string will always be in ISO 8601 format"
   - "This dictionary key will always exist"
   - "The parent view will always pass a non-nil value"
   - "This enum will never have new cases added"
3. For each assumption, determine:
   - **How likely is violation?** (Never / Rare / Occasional / Common)
   - **What happens on violation?** (Crash / Silent data loss / Wrong UI / Graceful degradation)
   - **Is there a guard?** (Yes — handled / No — unprotected)
4. Flag as BUG if: likelihood >= Rare AND consequence >= Silent data loss AND no guard

**Search accelerators** (grep for common assumption patterns):
```
Grep pattern="\.first!" glob="$SOURCE_GLOB"
Grep pattern="\.last!" glob="$SOURCE_GLOB"
Grep pattern="\[0\]" glob="$SOURCE_GLOB"
Grep pattern="\.removeFirst\(\)" glob="$SOURCE_GLOB"
Grep pattern="as!" glob="$SOURCE_GLOB"
```

### Lens 2: State Machine Analysis

**Goal:** Find states that become unreachable, states that can be simultaneously active, and transitions that leave the app in an inconsistent state.

**Process:**
1. Identify all `@State`, `@Published`, `@Observable` properties that represent user-facing state
2. Map the state transitions:
   - Draw the state graph: which states lead to which?
   - Can any state become a dead end? (entered but never exited)
   - Can two mutually exclusive states be true simultaneously?
3. Check interruption resilience:
   - What happens if the user backgrounds the app mid-transition?
   - What happens if a sheet is dismissed during an async operation?
   - What happens if the view disappears before a Task completes?
4. Check reset paths:
   - After an error, does state reset to a usable starting point?
   - After cancellation, are all intermediate states cleaned up?

**Search accelerators:**
```
Grep pattern="@State.*=.*true" glob="$SOURCE_GLOB"
Grep pattern="@Published.*=.*true" glob="$SOURCE_GLOB"
Grep pattern="isLoading|isProcessing|isSaving|isAnalyzing" glob="$SOURCE_GLOB"
Grep pattern="showingError|showingAlert|showingSheet" glob="$SOURCE_GLOB"
```

### Lens 3: Boundary Conditions

**Goal:** Find off-by-one errors, empty collection crashes, and edge case failures.

**Process:**
1. For every conditional (`if`, `guard`, `switch`), evaluate:
   - **Zero:** What happens with 0 items, empty string, nil?
   - **One:** What happens with exactly 1 item? (Singular/plural, first==last)
   - **Maximum:** What happens at Int.max, massive arrays, very long strings?
   - **Negative:** What happens with negative numbers where unsigned is expected?
2. For every array/collection access:
   - Is the index bounds-checked?
   - Does `removeFirst()` / `removeLast()` have an emptiness guard?
   - Does `prefix()` / `suffix()` handle count > array.count?
3. For every string operation:
   - Does it handle empty strings?
   - Does it handle Unicode (emoji, RTL, combining characters)?
   - Does it handle strings with special characters (quotes, backslashes)?

**Search accelerators:**
```
Grep pattern="\.remove(First|Last)\(\)" glob="$SOURCE_GLOB"
Grep pattern="\.\[.*\]" glob="$SOURCE_GLOB"
Grep pattern="\.prefix\(|\.suffix\(" glob="$SOURCE_GLOB"
Grep pattern="\.split\(|\.components\(" glob="$SOURCE_GLOB"
Grep pattern="Int\.max|\.count\s*-\s*1" glob="$SOURCE_GLOB"
```

### Lens 4: Data Lifecycle Tracing

**Goal:** Follow data from creation to deletion and find gaps where data can be lost, duplicated, or become stale.

**Process:**
1. Pick the primary data entities (models, cached values, user input)
2. For each entity, trace:
   - **Creation:** Where is it created? What validations exist at creation time?
   - **Modification:** Where is it modified? Can modifications conflict? Are changes persisted immediately or batched?
   - **Persistence:** Where is it saved? Can the save fail silently? Is there a race between saves?
   - **Display:** Where is it displayed? Can stale data be shown after a background update?
   - **Deletion:** Where is it deleted? Are all references cleaned up? Are related entities cascaded?
   - **Sync:** If synced, what happens on conflict? Is there a "last write wins" that could lose data?
3. Flag gaps:
   - Created but never persisted (in-memory only, lost on crash)
   - Modified but UI not updated (stale display)
   - Deleted but references remain (dangling pointers, orphaned data)
   - Synced but no conflict resolution

**Search accelerators:**
```
Grep pattern="context\.insert\(" glob="$SOURCE_GLOB"
Grep pattern="context\.delete\(" glob="$SOURCE_GLOB"
Grep pattern="context\.save\(" glob="$SOURCE_GLOB"
Grep pattern="\.deleteRule\s*=" glob="$SOURCE_GLOB"
Grep pattern="@Query" glob="$SOURCE_GLOB"
```

### Lens 5: Error Path Exerciser

**Goal:** For every try/catch, async/await, and optional chain — determine what happens when the error path is taken.

**Process:**
1. Find all error-producing code:
   - `try` / `try?` / `try!`
   - `await` (can throw via Task cancellation)
   - Optional chaining (`?.`)
   - `guard let ... else { return }`
2. For each error path, evaluate:
   - **Does the UI reflect the error?** Or does it show a loading state forever?
   - **Is the app left in a consistent state?** Or is it half-modified?
   - **Can the user retry?** Or are they stuck?
   - **Is the error logged?** Or silently swallowed?
3. Specifically check for:
   - `try?` that converts a meaningful error to nil (information loss)
   - `catch` blocks that log but don't update UI state
   - Missing `isLoading = false` in error paths (loading state trap)
   - `guard let` with bare `return` that silently aborts a user-initiated action
   - `Task {}` without error handling (errors vanish)

**Search accelerators:**
```
Grep pattern="try\?" glob="$SOURCE_GLOB"
Grep pattern="catch\s*\{" glob="$SOURCE_GLOB" -A 3
Grep pattern="guard let.*else\s*\{\s*return\s*\}" glob="$SOURCE_GLOB"
Grep pattern="Task\s*\{" glob="$SOURCE_GLOB" -A 5
```

### Lens 6: Time-Dependent Bugs

**Goal:** Find code that behaves differently based on time, speed, or duration.

**Process:**
1. **Timezone and locale:**
   - DateFormatters without explicit timezone set
   - Date comparisons using `<` instead of Calendar methods
   - String-to-date parsing that assumes a specific locale
   - Display of dates without considering user's locale
2. **Duration since last use:**
   - Cached data that expires but has no expiry check
   - Tokens/sessions that timeout but the app doesn't re-authenticate
   - "First launch" flags that don't account for app updates
3. **Rapid repeated actions:**
   - Buttons without debouncing that trigger duplicate network calls
   - Double-tap on save that creates duplicate records
   - Rapid navigation that crashes due to concurrent presentations
4. **Slow or absent network:**
   - Operations that assume network availability
   - Missing timeout on network requests
   - No offline fallback for features that need data
   - Optimistic UI that doesn't roll back on failure

**Search accelerators:**
```
Grep pattern="DateFormatter\(\)" glob="$SOURCE_GLOB"
Grep pattern="Date\(\)" glob="$SOURCE_GLOB"
Grep pattern="timeIntervalSince" glob="$SOURCE_GLOB"
Grep pattern="URLSession.*dataTask|URLSession.*data\(" glob="$SOURCE_GLOB"
Grep pattern="\.debounce|\.throttle" glob="$SOURCE_GLOB"
```

### Lens 7: Platform Divergence

**Goal:** Find code that assumes specific hardware, OS version, or device capabilities.

**Process:**
1. **Hardware assumptions:**
   - Image/video processing that assumes GPU acceleration (fails on Intel Macs, old devices)
   - Memory-intensive operations without pressure checks
   - Camera/sensor access without capability checks
   - Biometric auth that assumes Face ID (could be Touch ID or neither)
2. **OS version gaps:**
   - `#available` checks where the `else` branch is incomplete or empty
   - Features gated by availability but degraded path not tested
   - API behavioral differences between OS versions (not just availability)
3. **Device-specific behavior:**
   - Screen size assumptions (hardcoded widths, missing adaptive layout)
   - Keyboard presence assumptions (iPad with hardware keyboard, Mac)
   - Rotation handling that only works on certain devices
   - Split View / Stage Manager compatibility
4. **Platform #if blocks:**
   - `#if os(iOS)` with logic not mirrored in `#else` (macOS)
   - Platform-specific features without feature detection
   - Conditional compilation where one branch is clearly less tested

**Search accelerators:**
```
Grep pattern="#if os\(" glob="$SOURCE_GLOB"
Grep pattern="#available\(" glob="$SOURCE_GLOB"
Grep pattern="UIScreen\.main" glob="$SOURCE_GLOB"
Grep pattern="UIDevice\.current" glob="$SOURCE_GLOB"
Grep pattern="ProcessInfo.*physicalMemory|ProcessInfo.*processorCount" glob="$SOURCE_GLOB"
Grep pattern="CIContext|MTLDevice|MTLCreateSystemDefaultDevice" glob="$SOURCE_GLOB"
```

---

## Step 3: Verification Rule

Before classifying ANY finding:

1. **Read the flagged file** — at minimum 30 lines around the issue. Read enough to capture any enclosing `#if os(...)` block, even if that means widening the window beyond 30 lines.
2. **Check for existing guards** — the assumption may already be protected
3. **Check for intentional design** — comments, documentation, or clear architectural decisions
4. **Check if it's actually reachable** — dead code or unreachable paths are not bugs
5. **Assess real-world likelihood** — theoretical issues in impossible scenarios are not bugs
6. **Count blast radius** — grep for callers/references of the affected function or property to determine how many files the fix would touch
7. **Honor the platform context from Step 0.** If the match sits inside a `#if os(X)` block where X is NOT in the project's declared platform set, the finding is OK (excluded by build), not BUG. Example: on an iOS-only project, code inside `#if os(macOS)` is dead from the project's perspective and must not be flagged. On a Universal project, code inside `#if os(iOS)` is live for iOS but excluded for macOS — flag the finding only if the bug condition can occur on a platform that DOES include the block.
8. **Classify** as:
   - **BUG:** Real risk, no guard, realistic scenario, on a platform the project ships to
   - **FRAGILE:** Works now but will break under foreseeable conditions
   - **OK:** Already guarded, intentional design, OR excluded by `#if` conditional from the project's platform set
   - **REVIEW:** Unclear, needs human judgment

**Classification rules:**
- A finding is NOT a bug if the code already handles the case (even if the handling is in a different function)
- A finding is NOT a bug if the scenario requires conditions that the app's architecture prevents
- A finding is NOT a bug if it sits inside an `#if os(...)` block excluded from the project's platform set (Step 0)
- A finding IS a bug if a real user on a shipping platform could trigger it through normal usage
- A finding is FRAGILE if it works today but depends on an assumption that could change

---

## Step 4: Generate Report

### 4.1: Check Terminal Width

Before rendering the report, check terminal width:

```bash
tput cols
```

The full table is 10 columns (`#` through Confidence), plus 1-Star Risk when store-distributed and Status on re-display — so 10 to 12 depending on the run. Budget ~180 columns for the widest form. Count the pipes in the header below rather than trusting this sentence; it was wrong by one for four releases.

- **180+ columns:** Use the **full table** inline and in the report file.
- **Under 180 columns:** Use the **compact 4-column table** inline, write the **full table** to the report file only. Display this notice after the inline table:

> **Compact view** — your terminal is [N] columns wide (180+ needed for the full table). The complete Issue Rating Table with all [K] columns has been written to `.agents/research/YYYY-MM-DD-bug-prospector-<scope>.md`. Open that file or widen your terminal to 180+ columns to view it as a single table.

**If `tput cols` fails or returns exactly 80** — the default when output is not a TTY, which is the common case when this skill runs — the width is unknown, not narrow. Take the compact path and say why, in one line:

> Terminal width unknown (`tput cols` reported the non-TTY default). Showing the compact table inline; the full [K]-column table is in the report file.

Do not prompt for a width. The user cannot usefully answer a question about their column count, the full table is written to the report file on every path regardless, and a prompt on the common path taxes every run to change only which of two tables renders inline. If the user wants the full table inline, they can say so and you re-render it.

### 4.2: Render Report

**Display the summary and findings inline**, then write to `.agents/research/YYYY-MM-DD-bug-prospector-<scope>.md`.

The **report file** always uses the full table regardless of terminal width.

```markdown
# Bug Prospector Report: [Scope Description]

**Date:** YYYY-MM-DD
**Scope:** [Files/features analyzed]
**Lenses Applied:** [List of lenses used]
**Files Analyzed:** N

## Summary

| Status | Count |
|--------|-------|
| Bugs Found | X |
| Fragile Code | Y |
| OK (Already Guarded) | Z |
| Needs Review | W |

## Issue Rating Table

All BUG and FRAGILE findings rated and sorted by Urgency then ROI:
```

**Full table (180+ columns) — used inline AND in report file.** The 11th column is **1-Star Risk**, included ONLY when Step 0 recorded *store-distributed: yes*:

| # | Finding | Lens | Urgency | Risk: Fix | Risk: No Fix | ROI | Blast Radius | Fix Effort | Confidence | 1-Star Risk |
|---|---------|------|---------|-----------|-------------|-----|-------------|------------|------------|-------------|
| 1 | File.swift:45 — array accessed without bounds check | Boundary | 🔴 Critical | ⚪ Low | 🔴 Critical | 🟠 Excellent | ⚪ 1 file | Trivial | verified | 🔴 High |
| 2 | Manager.swift:89 — isLoading never reset on error | Error Path | 🟡 High | ⚪ Low | 🟡 High | 🟠 Excellent | ⚪ 1 file | Trivial | verified | 🟡 Medium |
| 3 | ViewModel.swift:200 — DateFormatter without timezone | Time | 🟢 Medium | ⚪ Low | 🟢 Medium | 🟢 Good | 🟢 3 files | Small | probable | 🟢 Low |

**When *store-distributed: no*, drop the 1-Star Risk column entirely** — do not emit it empty or filled with ⚪. A backend service or library has no review page; the column would be dead weight in every row.

**Status column (conditional, appended last).** Add a **Status** column (`Open` / `Fixed` / `Deferred` / `Skipped`) whenever the table is re-displayed after the initial report — during the fix workflow's phase gates, or in any follow-up run that revisits earlier findings. Omit it on first display, where every finding is `Open` by definition. With both conditional columns present the table runs to 12.

**Compact table (under 180 columns) — used inline only:**

| # | Finding | Urgency | Fix Effort |
|---|---------|---------|------------|
| 1 | File.swift:45 — array accessed without bounds check | 🔴 Critical | Trivial |
| 2 | Manager.swift:89 — isLoading never reset on error | 🟡 High | Trivial |
| 3 | ViewModel.swift:200 — DateFormatter without timezone | 🟢 Medium | Small |

The compact table keeps the two most actionable columns (Urgency + Fix Effort). Full ratings are in the report file.

```markdown
Use the Issue Rating scale:
- **Urgency:** 🔴 CRITICAL (crash/data loss) · 🟡 HIGH (incorrect behavior) · 🟢 MEDIUM (degraded UX) · ⚪ LOW (cosmetic/minor)
- **Risk: Fix:** Risk of the fix introducing regressions (⚪ Low for isolated changes, 🟡 High for shared code paths)
- **Risk: No Fix:** User-facing consequence if left unfixed
- **ROI:** 🟠 Excellent · 🟢 Good · 🟡 Marginal · 🔴 Poor
- **Blast Radius:** Number of files the fix touches (e.g., "⚪ 1 file", "🟢 3 files", "🟡 12 files"). Count by grepping for callers/references before rating.
- **Fix Effort:** Trivial / Small / Medium / Large
- **Confidence:** How the rating was reached. `verified` (code read and confirmed by grep or test) · `probable` (structural analysis, not independently verified) · `needs-runtime` (requires running the app to confirm). Always populated. A finding classified REVIEW is never `verified`.
- **1-Star Risk** *(only when store-distributed)*: How likely this finding, left unfixed, would provoke a one-star review — 🔴 High · 🟡 Medium · 🟢 Low · ⚪ Not scored. Rate from user-visible consequence, not code severity: data loss, lost work, and "the app lied to me" score high; a slow path or a cosmetic gap scores low. A finding can be 🔴 Critical on Urgency and 🟢 Low here (an internal crash on a path few users reach), or the reverse. Use ⚪ when you have not scored it rather than guessing a band.
```

> **On the 1-Star Risk band:** this is bug-prospector's own judgment from the finding's user-facing consequence, not output from a dedicated review-risk tool. Treat it as a prioritization hint among findings, not a measured prediction of store ratings.

## Implementation Plan

Phased fix order for the BUG and FRAGILE findings above. Emit this section whenever there is at least one BUG finding; omit it on a clean run.

Group by **file proximity and dependency**, then order phases by the highest Urgency each contains:

- Findings in the same file belong in the same phase — one read, one edit pass, one build.
- A finding whose fix changes a signature or shared type comes BEFORE any finding that depends on it, even if the dependent one is more urgent. Dependency order beats urgency order; note it when the two conflict.
- Keep a phase to one build-verify cycle's worth of work. A phase spanning 8 files is two phases.

```
### Phase 1: [Name] — [N fixes]
**Files:** path/to/File.swift, path/to/Other.swift
**Findings:** 1, 3, F1
**Rationale:** [why these group — same file, shared type, or blocking a later phase]
**Verify:** [what to build or run before moving on]
```

Phase numbering here is what Step 5's fix workflow walks through — the phase gate names the phase and counts its fixes from this section, so the two always agree.

## Detailed Findings

### 1. [File:Line] — [Brief Description]
**Lens:** [Which lens found this]
**Assumption:** [What the code assumes]
**Violation scenario:** [How a real user triggers this]
**Consequence:** [What happens — crash, data loss, wrong UI, etc.]

**Current code:**
```swift
// problematic code
```

**Suggested fix:**
```swift
// corrected code
```

### 2. [File:Line] — [Brief Description]
...

## Fragile Code (Works Now, May Break Later)

### F1. [File:Line] — [Brief Description]
**Lens:** [Which lens found this]
**Current behavior:** [What happens now]
**Breaking scenario:** [What foreseeable change would break this]
**Recommendation:** [How to harden it]

## Already Guarded (Reference)
These were flagged by search but are correctly handled:
- file.swift:123 — [why it's OK]

## Needs Human Review
These findings require domain knowledge to classify:
- file.swift:456 — [what's unclear and why]

## After You Fix These Findings

bug-prospector finds bug *patterns*. After you fix any BUG finding above, run [bug-echo](https://github.com/Terryc21/bug-echo) on the fix to find sibling instances of the same pattern elsewhere in the codebase:

```
/bug-echo
```

bug-echo infers the anti-pattern from your fix's diff, validates it against the pre-fix file, then scans for matches. Bugs that haven't fired yet but share the same shape are the highest-ROI catches in any audit cycle. Run bug-prospector to find the first instance; run bug-echo to close the bug class.
```

---

## Step 5: Follow-up

```
AskUserQuestion with questions:
[
  {
    "question": "How would you like to proceed?",
    "header": "Next",
    "options": [
      {"label": "Fix all bugs now", "description": "Walk through each BUG finding and apply fixes (phase-by-phase)"},
      {"label": "Fix selected bugs", "description": "I'll choose which ones to fix"},
      {"label": "Create implementation plan", "description": "Generate a phased plan from the findings"},
      {"label": "Report is sufficient", "description": "I'll handle fixes manually"}
    ],
    "multiSelect": false
  }
]
```

If creating plan: the report's **Implementation Plan** section (Step 4) already contains it — present that, expanded with per-finding fix notes. Do not derive a second, differently-grouped plan; two plans for one report is how the phase numbering drifts.

### Fix Workflow: "Fix all bugs now"

Walk the phases from the report's **Implementation Plan** section in order — that section is the source of truth for grouping and numbering, so Phase 2 here is Phase 2 there. Before each phase, prompt with a **phase gate** — one question, always includes opt-out:

```
AskUserQuestion with questions:
[
  {
    "question": "Phase N: [Phase name] ([count] fixes — [key files]). Proceed?",
    "header": "Phase N",
    "options": [
      {"label": "Proceed (Recommended)", "description": "Implement this phase now"},
      {"label": "Proceed — skip remaining phase gates", "description": "Batch mode: implement all remaining phases without prompting"},
      {"label": "Let's chat about this phase", "description": "Discuss before deciding"},
      {"label": "Stop here", "description": "Skip remaining phases"}
    ],
    "multiSelect": false
  }
]
```

**Every phase gets this same prompt** (including the opt-out) until the user either completes all phases or opts out.

**Rules:**
- If user chooses "skip remaining phase gates": execute all remaining phases without prompting
- If user chooses "Let's chat": wait for user input, answer questions, then re-prompt the same phase gate
- If user chooses "Stop here": stop fixing and report what was completed
- Always build both platforms after each phase to verify

### Fix Workflow: "Fix selected bugs"

Present the bug list for selection. **The list is generated from the actual findings — it is never a fixed three.** `AskUserQuestion` allows at most 4 options per question and 4 questions per call, so 16 findings fit in one call; beyond that, page.

**Build the options:** one option per BUG and FRAGILE finding, in the Issue Rating Table's order (Urgency, then ROI). Label each with its table row number so the two views line up — `Bug 1`, `Bug 2`, `F1`, `F2`. Description carries `[Urgency] [Fix Effort]`.

| Findings | Shape |
|---|---|
| 1–4 | One question, one call |
| 5–16 | 2–4 questions in ONE call, 4 options each — as Step 1.2 does for the 7 lenses |
| 17+ | Page: 16 per call, "Select from findings 1–16", then a second call for 17–32, and so on |

Split multi-question calls at natural boundaries where one exists — all 🔴 Critical in the first question, 🟡 High in the second — rather than mechanically at every fourth row. A user scanning for the critical ones should find them together.

Every question uses `"multiSelect": true`. **Selections accumulate across questions and across pages**: the user picks from question 1 AND question 2 AND page 2, and the union is the final set. Never treat a later question as replacing an earlier answer.

```
AskUserQuestion with questions:
[
  {
    "question": "Which bugs should I fix? (Reference the Issue Rating Table above)",
    "header": "Select",
    "options": [
      {"label": "Bug 1 — [brief description]", "description": "[Urgency] [Fix Effort]"},
      {"label": "Bug 2 — [brief description]", "description": "[Urgency] [Fix Effort]"},
      {"label": "Bug 3 — [brief description]", "description": "[Urgency] [Fix Effort]"},
      {"label": "Bug 4 — [brief description]", "description": "[Urgency] [Fix Effort]"}
    ],
    "multiSelect": true
  },
  {
    "question": "More findings:",
    "header": "More",
    "options": [
      {"label": "F1 — [brief description]", "description": "[Urgency] [Fix Effort]"},
      {"label": "F2 — [brief description]", "description": "[Urgency] [Fix Effort]"}
    ],
    "multiSelect": true
  }
]
```

The shape above is illustrative — 6 findings across 2 questions. Emit as many questions as the finding count requires, up to 4 per call.

🛑 **Never truncate the list to fit.** If a run produces 30 findings, page through all 30. Silently dropping findings 17+ presents a partial list as if it were complete, and the user has no way to see what is missing.

After selection, one confirmation prompt with opt-out:

```
AskUserQuestion with questions:
[
  {
    "question": "Fix [N] selected bugs: [brief list]. Proceed?",
    "header": "Confirm",
    "options": [
      {"label": "Proceed (Recommended)", "description": "Fix all selected bugs now"},
      {"label": "Let's chat first", "description": "Discuss specific bugs before fixing"},
      {"label": "Go back", "description": "Re-select bugs"}
    ],
    "multiSelect": false
  }
]
```

Keep `[brief list]` readable: name up to 5 selected findings by row number, then `+N more` (`Bug 1, Bug 2, Bug 4, F1, F2 +12 more`). The full list was just on screen during selection; the confirmation is a sanity check on the count, not a re-read.

Then fix all selected bugs without per-bug prompting — the selection itself was the decision.

**If user chooses "Let's chat":** Answer questions, then re-prompt the confirmation.

**If the user selects nothing** (all questions dismissed or no options chosen): do not fall through to fixing everything. Say that nothing was selected and re-prompt the selection once; if still empty, stop and leave the report as the deliverable.

---

## Lens Selection Guide

| Situation | Recommended Lenses |
|-----------|--------------------|
| Pre-release audit | All 7 |
| After adding a new feature | 1 (Assumptions) + 2 (State) + 5 (Errors) |
| After a crash report | 3 (Boundaries) + 5 (Errors) + 7 (Platform) |
| Debugging intermittent failures | 2 (State) + 6 (Time) |
| New platform support (macOS/iPad) | 7 (Platform) + 3 (Boundaries) |
| Data model changes | 4 (Data Lifecycle) + 1 (Assumptions) |
| Performance investigation | 3 (Boundaries) + 6 (Time) |

---

## What This Finds vs. What Auditors Find

| Auditors Find | Bug Prospector Finds |
|---------------|---------------------|
| Missing `@MainActor` | State machine deadlocks |
| Force unwraps | Assumptions that hold today but break tomorrow |
| Retain cycles | Data that's created but never cleaned up |
| Missing `try?` conversion | Error paths that leave UI in loading state |
| Deprecated API usage | Code that works on Apple Silicon but fails on Intel |
| Missing accessibility labels | Rapid-tap scenarios that create duplicate data |

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Too many findings | Narrow scope to specific files or use "Quick 3" lenses |
| All findings are theoretical | Increase the realism threshold — only flag scenarios a real user could trigger |
| Overlapping with auditor findings | Focus on *why* not *what* — auditors find the pattern, prospector finds the consequence |
| File too complex to analyze | Split analysis by MARK sections or extract to Agent subprocesses |
| Can't determine if guarded | Classify as REVIEW, not BUG — let the human decide |
