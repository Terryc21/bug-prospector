# bug-prospector

![Version](https://img.shields.io/github/v/tag/Terryc21/bug-prospector?label=version) ![Last commit](https://img.shields.io/github/last-commit/Terryc21/bug-prospector) ![Stars](https://img.shields.io/github/stars/Terryc21/bug-prospector?style=flat) ![Issues](https://img.shields.io/github/issues/Terryc21/bug-prospector) ![License](https://img.shields.io/github/license/Terryc21/bug-prospector) ![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-Plugin-blueviolet)

**Your code compiles. Tests pass. It looks right. Then a real user does something you didn't think of, and it breaks.**

bug-prospector is a Claude Code skill that goes looking for those bugs before the user does. It reads your code and asks what each piece assumes — then works out what happens when the assumption turns out to be wrong.

> **Companion:** [bug-echo](https://github.com/Terryc21/bug-echo) — runs *after* a fix to find sibling instances of the same bug pattern. The two skills cover opposite halves of the bug-finding loop.

Built while shipping [Stuffolio](https://stuffolio.app), an iOS/macOS app, through real App Store submission cycles. bug-prospector itself is free, open source, Apache 2.0.

*~5 min read · [how it works](docs/HOW_IT_WORKS.md) · [an example report](skills/bug-prospector/examples/2026-04-29-bug-prospector-backup-manager.md) · [what changed](https://github.com/Terryc21/bug-prospector/releases)*

## TL;DR

- **What:** 7 forward-looking analysis lenses (assumptions, state machines, boundaries, lifecycle, errors, time, platform) that find behavioral bugs in code that compiles fine and passes tests.
- **Why:** Linters find code that *looks* wrong (force unwraps, deprecated APIs, retain cycles). bug-prospector finds code that *looks right but behaves wrong* — quiet assumptions that turn out not to hold under real-world conditions.
- **Install:** Two `/plugin` commands in Claude Code; then `/bug-prospector` is available in any project.
- **Try first:** `/bug-prospector quick` runs 3 of 7 lenses (Assumptions + Errors + Boundaries) on your recent changes. Light, fast, real report.
- **Example output:** [a real backup-manager audit on Stuffolio](skills/bug-prospector/examples/2026-04-29-bug-prospector-backup-manager.md) (4 BUG findings, 4 lenses). Also: [quick-scan on TypeScript](skills/bug-prospector/examples/quick-scan-auth-middleware.md).
- **Maturity:** Used through real App Store submission cycles. Swift/SwiftUI is where it's deepest; other languages work, with a caveat in Honest limits below.

## The kind of bug this finds

- What happens when that array is empty?
- What if the user double-taps Save before the first save finishes?
- What if the network call completes after the view disappears?
- What if they open the app for the first time in 3 months?

None of those is wrong on any single line, which is why linters don't catch them. Linters check *how your code is written* — force unwraps, deprecated APIs, retain cycles. bug-prospector checks *what your code assumes*.

A linter is the building inspector confirming every wire is up to code. bug-prospector is the home inspector asking what happens when the dishwasher and the microwave run at the same time. Different layer, different bugs — run both. ([The longer comparison](docs/HOW_IT_WORKS.md), if you want it.)

## bug-prospector vs. bug-echo — which should you use?

Both have "bug" in the name; they answer different questions and run at different times.

| | bug-prospector | [bug-echo](https://github.com/Terryc21/bug-echo) |
|---|---|---|
| **When you run it** | Before a release, after a crash report, during exploration | Right after you fix a bug |
| **What it asks** | "What could go wrong?" | "Where else does this exact thing live?" |
| **What it needs** | Just code | The diff of a fix you just committed |
| **Pattern source** | 7 forward-looking lenses (assumptions, state machines, boundaries, lifecycle, errors, time, platform) | The pattern is inferred from your actual diff and validated against the pre-fix file |

Many people run both — bug-prospector before releases, bug-echo after every bug fix. They complement each other.

## Install

Two commands in Claude Code, run one at a time:

```
/plugin marketplace add Terryc21/bug-prospector
```

```
/plugin install bug-prospector@bug-prospector
```

> **Why two commands?** Claude Code's slash-command dispatcher treats the second `/plugin` as text inside the first command. Run them one at a time and wait for the first to confirm before running the second.

After installing, try:

```
/bug-prospector quick
```

This runs three of the seven analysis passes (Assumptions, Errors, Boundaries) on your recent changes. It's the lightest run bug-prospector offers — produces a real report you can act on without committing to the full sweep.

### Optional: install bug-echo alongside

bug-prospector runs before a fix. [bug-echo](https://github.com/Terryc21/bug-echo) runs after one — same workflow loop, opposite end. Most users want both:

```
/plugin marketplace add Terryc21/bug-echo
/plugin install bug-echo@bug-echo
```

(Same one-at-a-time rule applies.)

## Your first runs

```
/bug-prospector quick
```

Runs three lenses (Assumptions, Errors, Boundaries) on your recent changes. Light, fast, low token cost. Good for a first run or for routine pre-PR audits.

```
/bug-prospector all
```

Runs all 7 lenses. Heavier, more findings, takes longer. Save it for before a release or after a crash report.

```
/bug-prospector
```

No arguments — prompts you to choose scope and lenses interactively. Useful when you want to focus on one area of your code.

## What it looks for (the 7 lenses)

Each lens is a different angle for asking "what could go wrong here?"

| # | Lens | What it finds |
|---|------|---------------|
| 1 | **Assumptions** | Implicit assumptions that hold today but break when conditions change |
| 2 | **State machine** | States that shouldn't be reachable, two states active at once, transitions that get interrupted |
| 3 | **Boundary conditions** | Zero, one, the maximum value, an empty collection, off-by-one errors |
| 4 | **Data lifecycle** | Data created but never cleaned up, stale displays, orphaned references |
| 5 | **Error paths** | Errors that leave the UI stuck, errors swallowed silently, errors that lose user data |
| 6 | **Time-dependent bugs** | Timezones, rapid taps creating duplicates, slow networks, first-launch-after-weeks code |
| 7 | **Platform divergence** | Code that works on Apple Silicon but fails on Intel, or on iOS but not macOS, OS-version gaps |

You usually don't need all 7 every time. The right combination depends on what you just changed:

| Situation | Useful lenses |
|---|---|
| Pre-release audit | All 7 |
| After adding a new feature | Assumptions + State machine + Error paths |
| After a crash report | Boundaries + Error paths + Platform |
| Debugging intermittent failures | State machine + Time-dependent |
| Adding support for a new platform | Platform + Boundaries |
| Changing your data model | Data lifecycle + Assumptions |

## Scoping a run

Two dimensions, picked independently: *what to look at* (recent changes, one file, the whole codebase) and *which lenses to apply* (quick 3, all 7, or a subset you choose). Add a path to either preset to scope it: `/bug-prospector quick src/auth.ts`.

**Every run is fresh.** The skill reads your current code from scratch — there's no resume mode and no diff-against-last-report. Old reports stay in `.agents/research/` if you want to compare by hand. If a finding keeps coming back and you've decided it's a false positive for your codebase, say so in your project's CLAUDE.md; the skill reads that when classifying.

## Output format

Every run writes a rated table to `.agents/research/YYYY-MM-DD-bug-prospector-*.md`. Three real rows from the sample report:

| # | Finding | Lens | Urgency | ROI | Fix Effort | Confidence |
|---|---------|------|---------|-----|------------|------------|
| 1 | `BackupManager.swift:1414` — `try? rollback()` swallows rollback failure during restore | Error Path | 🔴 Critical | 🟠 Excellent | Trivial | verified |
| 2 | `BackupManager.swift:1216` — 8 fetches via `try? context.fetch` silently return empty arrays | Error Path | 🟡 High | 🟠 Excellent | Small | verified |
| F1 | `ItemListViewModel.swift:445` — filter rebuild races when filters change rapidly | Time | 🟢 Medium | 🟡 Marginal | Small | needs-runtime |

Every row carries a file and line, so you can go straight to the code. **Confidence** says how the rating was reached — `verified` means the code was read and confirmed, `needs-runtime` means you'd have to run the app to be sure. Two more columns appear when they apply: **1-Star Risk** on app-store projects, **Status** once you start fixing.

Below the table, each finding gets the current code, a suggested fix, and the scenario that triggers it. Separate sections cover code that works now but will break later, and cases the lens checked and found already handled.

The full sample — 4 BUG, 2 FRAGILE, 3 OK, 1 REVIEW across 4 lenses: [example output](skills/bug-prospector/examples/2026-04-29-bug-prospector-backup-manager.md).

A lighter second example showing the **Quick 3** preset on a single TypeScript file: [quick-scan example](skills/bug-prospector/examples/quick-scan-auth-middleware.md).

The report doesn't change your code. You decide which findings to fix.

### Reading the reports

**The skill adapts to your terminal.** 180+ columns gets the full table inline; anything narrower gets a compact 4-column version. **The full table goes to the report file either way** — a compact inline view means fewer columns on screen, never fewer findings.

The table wants ~180 chars to render horizontally, so if it looks broken in your terminal, open the report file on GitHub, in VS Code's preview (cmd-shift-V), or any markdown viewer.

## Fixing what it finds

After the report, the skill offers four ways to proceed:

- **Fix all bugs now** — walks through each phase with a confirmation before each one. Opt out of remaining confirmations whenever you want.
- **Fix selected bugs** — pick specific findings, confirm once, all selected get fixed.
- **Create implementation plan** — phased plan without making any code changes.
- **Report is sufficient** — just the report. You take it from there.

## Honest limits

This skill is a tool, not an oracle. A few things to keep in mind:

- It surfaces candidates, not verdicts. The lens flags places where an assumption could break; you decide whether the assumption actually matters.
- False positives happen. A lens may flag a state-machine concern in code that's intentionally simple.
- False negatives happen. Bugs whose pattern doesn't fit any of the 7 lenses won't get caught.
- It can't run your code. It reads your code and reasons about it. Some bugs only show up at runtime; static reasoning has limits.
- **The lenses are language-neutral; the plumbing is Swift-tagged.** The search shortcuts and the scope picker both filter on `*.swift`. On another language, hand it a path — `/bug-prospector quick src/auth.ts` — and the lenses work normally. Let it pick the scope itself and it will find nothing and say so, rather than reporting a clean run. Widening this is on the list.

Treat findings as leads to investigate, not items to fix blindly. Verify critical findings before committing.

**Where to look for the bugs bug-prospector won't find:** pattern-based linters (SwiftLint, etc.) catch the single-file style violations; [bug-echo](https://github.com/Terryc21/bug-echo) catches sibling instances after a fix; runtime profiling (Instruments, sanitizers) catches concurrency and memory issues; targeted unit tests catch business-logic correctness. bug-prospector covers the forward-looking-assumptions slot in that picture.

## Deeper documentation

The full methodology lives in [docs/HOW_IT_WORKS.md](docs/HOW_IT_WORKS.md), including how bug-prospector pairs with [Workflow Audit](https://github.com/Terryc21/workflow-audit). ([README-detailed.md](README-detailed.md) is retired — it duplicated this file and drifted; it now just points here.)

## Sibling skills

- [**bug-echo**](https://github.com/Terryc21/bug-echo) — sibling-bug scan after a fix; companion skill
- [**workflow-audit**](https://github.com/Terryc21/workflow-audit) — 5-layer SwiftUI behavioral flow audit
- [**unforget**](https://github.com/Terryc21/unforget) — one-file deferred-work ledger
- [**radar-suite**](https://github.com/Terryc21/radar-suite) — 6-skill suite tracing user behavior paths through the app (iOS + macOS)
- [**prompter**](https://github.com/Terryc21/prompter) — prompt rewriting before execution
- [**skill-reviewer**](https://github.com/Terryc21/skill-reviewer) — candid reviews of other Claude Code skills
- [**tutorial-creator**](https://github.com/Terryc21/tutorial-creator) — annotated tutorials from your codebase
- [**sitrep**](https://github.com/Terryc21/sitrep) — Xcode workflow skills (formerly `xcode-workflow-skills`)

## Author

Terry Nyberg, [Coffee & Code LLC](https://stuffolio.app/). If bug-prospector catches a real bug for you, [a coffee](https://buymeacoffee.com/stuffolio) is appreciated. Issue reports about what worked or didn't are more useful.

[![Buy Me A Coffee](https://img.shields.io/badge/Buy%20Me%20A%20Coffee-FFDD00?style=flat&logo=buy-me-a-coffee&logoColor=black)](https://www.buymeacoffee.com/stuffolio)

## License

Apache 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
