# Bug Prospector

Mine for hidden bugs that pattern-based auditors miss.

## What It Does

**Code auditors** check *how your code is written* — they find force unwraps, missing `@MainActor`, retain cycles, and deprecated APIs. If the syntax breaks a rule, they flag it.

**Bug Prospector** checks *what your code assumes* — it finds logic that compiles and runs fine today but breaks when a real user does something unexpected. Things like:

- What happens when that array is empty?
- What if the user double-taps Save before the first save finishes?
- What if the network call completes after the view disappears?
- What if they open the app for the first time in 3 months?

**Auditors find code that looks wrong. Bug Prospector finds code that looks right but behaves wrong.**

## 7 Analysis Lenses

| # | Lens | What It Finds |
|---|------|---------------|
| 1 | **Assumption Audit** | Implicit assumptions that break under real-world conditions |
| 2 | **State Machine Analysis** | Unreachable states, simultaneous states, interrupted transitions |
| 3 | **Boundary Conditions** | Zero/one/max values, empty collections, off-by-one errors |
| 4 | **Data Lifecycle Tracing** | Data created but never cleaned up, stale displays, orphaned references |
| 5 | **Error Path Exerciser** | Error paths that leave UI stuck, swallow errors, or lose user data |
| 6 | **Time-Dependent Bugs** | Timezone issues, rapid-tap duplicates, slow network, first-launch-after-weeks |
| 7 | **Platform Divergence** | Code that works on Apple Silicon but fails on Intel, OS version gaps |

## Install

```bash
claude plugin add Terryc21/bug-prospector
```

Or clone and install locally:

```bash
git clone https://github.com/Terryc21/bug-prospector.git
claude plugin install ./bug-prospector
```

## Usage

```
/bug-prospector                    # Interactive — choose scope and lenses
/bug-prospector all                # Full 7-lens scan on recent changes
/bug-prospector quick              # Quick 3 lenses (Assumptions + Errors + Boundaries)
```

## Output

Reports are written to `.agents/research/YYYY-MM-DD-bug-prospector-*.md` with:
- Issue Rating Table (Urgency, Risk, ROI, Blast Radius, Fix Effort)
- Detailed findings with current code and suggested fixes
- Fragile code warnings (works now, may break later)
- Already-guarded instances (reference)

## When to Use

| Situation | Recommended Lenses |
|-----------|--------------------|
| Pre-release audit | All 7 |
| After adding a new feature | Assumptions + State + Errors |
| After a crash report | Boundaries + Errors + Platform |
| Debugging intermittent failures | State + Time |
| New platform support | Platform + Boundaries |
| Data model changes | Data Lifecycle + Assumptions |

## License

MIT
