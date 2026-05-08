# bug-prospector

**A Claude Code skill that finds bugs in code that compiles fine, passes tests, and looks right but breaks the moment a real user does something unexpected.**

> **Companion:** [bug-echo](https://github.com/Terryc21/bug-echo) — runs *after* a fix to find sibling instances of the same bug pattern. The two skills cover opposite halves of the bug-finding loop.

Built while shipping [Stuffolio](https://stuffolio.app), an iOS/macOS app I work on every day. Free, open source, no paid tier, no referral links.

<a href="https://buymeacoffee.com/stuffolio"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" width="120"></a>

If bug-prospector catches a real bug for you, a [coffee](https://buymeacoffee.com/stuffolio) is appreciated. Issue reports about what worked or didn't are even more useful.

---

## What is this, and why might I want it?

If you're newer to Claude Code and unsure what an "audit skill" does, here's the short version.

A **skill** is a markdown file Claude Code knows how to run. When you type `/bug-prospector`, Claude follows the instructions in that skill, looks at your code, and writes you a report. You don't have to memorize anything. The skill tells Claude what to do; you read the report.

Most code-checking tools (linters, compilers, static analyzers) check *how your code is written*. They flag force unwraps, missing `@MainActor`, retain cycles, deprecated APIs. If the syntax breaks a known rule, they catch it.

bug-prospector checks something different. It checks *what your code assumes*. Things like:

- What happens when that array is empty?
- What if the user double-taps Save before the first save finishes?
- What if the network call completes after the view disappears?
- What if they open the app for the first time in 3 months?

These are bugs that compile fine, pass tests, and look right. The bug isn't in any single line. It's in a quiet assumption that turns out not to hold under real-world conditions.

The short version: **linters find code that looks wrong. bug-prospector finds code that looks right but behaves wrong.**

---

## bug-prospector vs. bug-echo — which should you use?

Both have "bug" in the name; they answer different questions and run at different times.

| | bug-prospector | [bug-echo](https://github.com/Terryc21/bug-echo) |
|---|---|---|
| **When you run it** | Before a release, after a crash report, during exploration | Right after you fix a bug |
| **What it asks** | "What could go wrong?" | "Where else does this exact thing live?" |
| **What it needs** | Just code | The diff of a fix you just committed |
| **Pattern source** | 7 forward-looking lenses (assumptions, state machines, boundaries, lifecycle, errors, time, platform) | The pattern is inferred from your actual diff and validated against the pre-fix file |

Many people run both — bug-prospector before releases, bug-echo after every bug fix. They complement each other.

---

## Install

Two commands in Claude Code:

```
/plugin marketplace add Terryc21/bug-prospector
```

```
/plugin install bug-prospector@bug-prospector
```

Run them one at a time and wait for the first to confirm before running the second.

That's it. The skill is now available everywhere you use Claude Code.

### Optional: install bug-echo alongside

bug-prospector runs before a fix. [bug-echo](https://github.com/Terryc21/bug-echo) runs after one — same workflow loop, opposite end. Most users want both:

```
/plugin marketplace add Terryc21/bug-echo
```

```
/plugin install bug-echo@bug-echo
```

After installing, run `/bug-prospector` before releases (forward-looking audit) and `/bug-echo` after each bug fix (sibling scan).

---

## Your first run (start here)

You don't need to set anything up. Just run:

```
/bug-prospector quick
```

That runs three of the seven analysis passes ("lenses"): Assumptions, Errors, and Boundaries. It's the lightest run bug-prospector offers. It scans recent changes in your project, produces a report, and stays out of your weekly Claude Code allocation.

When you're comfortable with the format and want a deeper sweep, run:

```
/bug-prospector all
```

That runs all 7 lenses. Heavier, more findings, takes longer. Save it for before a release or after a crash report.

There's also `/bug-prospector` with no arguments, which prompts you to choose scope and lenses interactively. Useful when you want to focus on one area of your code.

---

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

---

## What the output looks like

Reports go to `.agents/research/YYYY-MM-DD-bug-prospector-*.md` in your project. Each report has:

- An issue rating table (urgency, risk, ROI, blast radius, fix effort)
- Detailed findings with the current code and a suggested fix
- A "FRAGILE" section for code that works now but is likely to break later
- An "Already guarded" section for cases the lens checked and confirmed are fine

A complete sample report (4 BUG findings, 2 FRAGILE, 3 OK, 1 REVIEW across 4 lenses) is here: [example output](skills/bug-prospector/examples/2026-04-29-bug-prospector-backup-manager.md).

A lighter second example showing the **Quick 3** preset on a single TypeScript file (an auth middleware audit before opening a PR): [quick-scan example](skills/bug-prospector/examples/quick-scan-auth-middleware.md). Useful for understanding the focused, single-file workflow alongside the heavier multi-lens audit.

The report doesn't change your code. You decide which findings to fix.

---

## Fixing what it finds

After the report, the skill offers four ways to proceed:

- **Fix all bugs now** — walks through each phase with a confirmation before each one. Opt out of remaining confirmations whenever you want.
- **Fix selected bugs** — pick specific findings, confirm once, all selected get fixed.
- **Create implementation plan** — phased plan without making any code changes.
- **Report is sufficient** — just the report. You take it from there.

---

## Why this is different from a linter

A linter checks each file against a list of patterns: "force unwraps are bad, unused variables are bad, missing `@MainActor` is bad." It's mechanical and fast.

bug-prospector asks behavioral questions: "if a user opens this screen for the first time in three months, does the trial timer still work? if they double-tap Save, does the second tap clobber the first?" Those questions don't have a code signature. The skill goes through them lens by lens, checking your code against each one.

A useful analogy: a linter is the building inspector confirming every wire is up to code. bug-prospector is the home inspector asking "what happens if the dishwasher and the microwave are running at the same time?"

---

## Honest about limits

This skill is a tool, not an oracle. A few things to keep in mind:

- It surfaces candidates, not verdicts. The lens flags places where an assumption could break; you decide whether the assumption actually matters.
- False positives happen. A lens may flag a state-machine concern in code that's intentionally simple.
- False negatives happen. Bugs whose pattern doesn't fit any of the 7 lenses won't get caught.
- It can't run your code. It reads your code and reasons about it. Some bugs only show up at runtime; static reasoning has limits.

The right way to use any audit skill: treat findings as leads to investigate, not items to fix blindly. Verify critical findings before committing.

---

## Other Claude Code skills I've built

- [tutorial-creator](https://github.com/Terryc21/tutorial-creator) — turns a file from your project into an annotated tutorial with vocabulary, quizzes, and gap analysis. Works for any language.
- [prompter](https://github.com/Terryc21/prompter) — rewrites your Claude Code prompt for clarity and fixes typos before acting.
- [bug-echo](https://github.com/Terryc21/bug-echo) — after you fix a bug, scans the codebase for sibling instances of the same pattern.
- [workflow-audit](https://github.com/Terryc21/workflow-audit) — 5-layer audit of SwiftUI user flows. Finds dead ends, dismiss traps, and unwired features.
- [radar-suite](https://github.com/Terryc21/radar-suite) — 6 audit skills for iOS/macOS Swift codebases. Covers data models, time-bomb code, navigation, backup/restore, visual quality.

All free, all Apache 2.0, all built while shipping Stuffolio.

---

## Deeper documentation

The previous, more detailed README is preserved as [README-detailed.md](README-detailed.md). The full methodology lives in [docs/HOW_IT_WORKS.md](docs/HOW_IT_WORKS.md), including how bug-prospector pairs with [Workflow Audit](https://github.com/Terryc21/workflow-audit).

---

## License

Apache 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).

## Author

Terry Nyberg, [Coffee & Code LLC](https://stuffolio.app/).
