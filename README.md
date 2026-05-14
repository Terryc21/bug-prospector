# bug-prospector

![Last commit](https://img.shields.io/github/last-commit/Terryc21/bug-prospector) ![Stars](https://img.shields.io/github/stars/Terryc21/bug-prospector?style=flat) ![Issues](https://img.shields.io/github/issues/Terryc21/bug-prospector) ![License](https://img.shields.io/github/license/Terryc21/bug-prospector) ![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-Plugin-blueviolet)

**A Claude Code skill that finds bugs in code that compiles fine, passes tests, and looks right but breaks the moment a real user does something unexpected.**

> **Companion:** [bug-echo](https://github.com/Terryc21/bug-echo) — runs *after* a fix to find sibling instances of the same bug pattern. The two skills cover opposite halves of the bug-finding loop.

bug-prospector and pattern-based linters are complementary, not competitive: linters check *how your code is written* against a rule catalog; bug-prospector checks *what your code assumes* against 7 forward-looking lenses. **A thorough audit uses both.**

Built while shipping [Stuffolio](https://stuffolio.app), an iOS/macOS app currently at build 33. Free, open source, Apache 2.0.

## TL;DR

- **What:** 7 forward-looking analysis lenses (assumptions, state machines, boundaries, lifecycle, errors, time, platform) that find behavioral bugs in code that compiles fine and passes tests.
- **Why:** Linters find code that *looks* wrong (force unwraps, deprecated APIs, retain cycles). bug-prospector finds code that *looks right but behaves wrong* — quiet assumptions that turn out not to hold under real-world conditions.
- **Install:** Two `/plugin` commands in Claude Code; then `/bug-prospector` is available in any project.
- **Try first:** `/bug-prospector quick` runs 3 of 7 lenses (Assumptions + Errors + Boundaries) on your recent changes. Light, fast, real report.
- **Example output:** [a real backup-manager audit on Stuffolio](skills/bug-prospector/examples/2026-04-29-bug-prospector-backup-manager.md) (4 BUG findings, 4 lenses). Also: [quick-scan on TypeScript](skills/bug-prospector/examples/quick-scan-auth-middleware.md).
- **Maturity:** Used through real App Store submission cycles; works on any language, with extra Swift/SwiftUI lens depth.

## What is this, and why might I want it?

If you're newer to Claude Code and unsure what an "audit skill" does, here's the short version.

A **skill** is a markdown file Claude Code knows how to run. When you type `/bug-prospector`, Claude follows the instructions in that skill, looks at your code, and writes you a report. You don't have to memorize anything. The skill tells Claude what to do; you read the report.

## What bug-prospector is for vs what linters are for

bug-prospector and linters are complementary — they catch different bugs at different layers.

**Pattern-based tools (SwiftLint, ESLint, custom rule sets, single-file linters)** check *how your code is written*. They flag force unwraps, missing `@MainActor`, retain cycles, deprecated APIs. If the syntax breaks a known rule, they catch it. They're fast, run continuously, and catch a real class of bugs cheaply. **They will find issues bug-prospector won't** — anything expressible as a single-file pattern.

**bug-prospector** checks *what your code assumes*. Things like:

- What happens when that array is empty?
- What if the user double-taps Save before the first save finishes?
- What if the network call completes after the view disappears?
- What if they open the app for the first time in 3 months?

These are bugs that compile fine, pass tests, and look right. The bug isn't in any single line. It's in a quiet assumption that turns out not to hold under real-world conditions. **bug-prospector will find issues linters won't** — assumption bugs have no code signature to search for.

The short version: **linters find code that looks wrong. bug-prospector finds code that looks right but behaves wrong.**

| What linters do better | What bug-prospector does better |
|---|---|
| Run on every save (cheap, continuous) | Run before release or after a crash (deeper, slower) |
| Catch style and pattern violations | Catch behavioral and assumption bugs |
| Mechanical, deterministic | Asks questions the rule catalog doesn't cover |
| Mature ecosystem | 7-lens forward-looking analysis |

A useful analogy: a linter is the building inspector confirming every wire is up to code. bug-prospector is the home inspector asking "what happens if the dishwasher and the microwave are running at the same time?" Different layer, different bugs. The inspection isn't complete without both.

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

## Output format

Reports go to `.agents/research/YYYY-MM-DD-bug-prospector-*.md` in your project. Standard format across the radar/audit ecosystem:

- File and line citations for every claim
- 9-column rating table: severity, urgency, risk-of-fix, risk-of-no-fix, ROI, blast radius, fix effort, status, axis classification
- 3-axis classification: Axis 1 (release-blocking), Axis 2 (quality), Axis 3 (hygiene)
- Detailed findings with the current code and a suggested fix
- A "FRAGILE" section for code that works now but is likely to break later
- An "Already guarded" section for cases the lens checked and confirmed are fine

A complete sample report (4 BUG findings, 2 FRAGILE, 3 OK, 1 REVIEW across 4 lenses): [example output](skills/bug-prospector/examples/2026-04-29-bug-prospector-backup-manager.md).

A lighter second example showing the **Quick 3** preset on a single TypeScript file: [quick-scan example](skills/bug-prospector/examples/quick-scan-auth-middleware.md).

The report doesn't change your code. You decide which findings to fix.

### Reading the reports

The 9-column rating table needs a wide terminal (~180 chars) to render as a horizontal table. In a narrower window the cells stack vertically and the report becomes harder to scan. For best readability:

- **GitHub or GitLab**: open the report file in the web UI; tables render natively.
- **Markdown viewer apps**: [MacDown](https://macdown.uranusjr.com/) (Mac, free), [Marked 2](https://marked2app.com/) (Mac, paid), [Obsidian](https://obsidian.md/) or [Typora](https://typora.io/) (cross-platform).
- **VS Code**: built-in Markdown Preview (cmd-shift-V on Mac).

If tables look broken in your terminal, widen the window or use one of the apps above.

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

Treat findings as leads to investigate, not items to fix blindly. Verify critical findings before committing.

**Where to look for the bugs bug-prospector won't find:** pattern-based linters (SwiftLint, etc.) catch the single-file style violations; [bug-echo](https://github.com/Terryc21/bug-echo) catches sibling instances after a fix; runtime profiling (Instruments, sanitizers) catches concurrency and memory issues; targeted unit tests catch business-logic correctness. bug-prospector covers the forward-looking-assumptions slot in that picture.

## Other Claude Code skills

Companion tools built on the same shipping-real-software loop:

- [**bug-echo**](https://github.com/Terryc21/bug-echo) — runs *after* a fix; sibling-instance scan. Companion skill.
- [**radar-suite**](https://github.com/Terryc21/radar-suite) — 6 audit skills for iOS/macOS Swift codebases. Covers data models, time-bomb code, navigation, backup/restore, visual quality, plus a capstone aggregator.
- [**workflow-audit**](https://github.com/Terryc21/workflow-audit) — 5-layer audit of SwiftUI user flows. Finds dead ends, dismiss traps, and unwired features.
- [**tutorial-creator**](https://github.com/Terryc21/tutorial-creator) — turns a file from your project into an annotated tutorial with vocabulary, quizzes, and gap analysis. Works for any language.
- [**unforget**](https://github.com/Terryc21/unforget) — consolidates deferred work into one structured file.
- [**prompter**](https://github.com/Terryc21/prompter) — rewrites your Claude Code prompt for clarity and fixes typos before acting.

All free, all Apache 2.0, all built while shipping Stuffolio.

## Deeper documentation

The previous, more detailed README is preserved as [README-detailed.md](README-detailed.md). The full methodology lives in [docs/HOW_IT_WORKS.md](docs/HOW_IT_WORKS.md), including how bug-prospector pairs with [Workflow Audit](https://github.com/Terryc21/workflow-audit).

## License

Apache 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).

## Author

Terry Nyberg, [Coffee & Code LLC](https://stuffolio.app/). If bug-prospector catches a real bug for you, [a coffee](https://buymeacoffee.com/stuffolio) is appreciated. Issue reports about what worked or didn't are even more useful.
