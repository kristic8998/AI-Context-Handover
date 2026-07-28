# OmniQuant AI — AI Handover Document

**First-hand.** Written by the architect who built the application, in the same
five-section format as the other documents in this repository.

**Repo:** `omniquant-ai` · v1.0.0 · 84 tests · 10-check selftest · CI green locally

**Provenance note:** the GUI has **never been rendered on a real screen.** The
build environment has no `tkinter`, so every UI file was verified through a mock Tk
layer that confirms construction and lifecycle but proves nothing about layout,
focus order, DPI behaviour or visual fit. Treat all seven pages as
layout-unverified. Everything below the UI — all engines, all statistics — was run
against real data and is verified numerically.

---

## 1. Core Purpose & Business Value

An analytics workbench for a senior data analyst at a digital micro-lending
company, replacing spreadsheet work that is either laborious or quietly wrong.

**Recovery Prophet** scores delinquent accounts for repayment likelihood
(XGBoost), estimates the day payment will land (Kaplan-Meier + Cox proportional
hazards), and sizes the late-fee waiver trade-off.

**Recon Sentinel** reconciles internal records against payment-gateway
settlements and the bank statement, using Levenshtein fuzzy matching for mangled
reference numbers and an Isolation Forest for anomalies.

**Causal Impact** tests whether a recovery campaign actually worked, using Welch's
t-test or a two-proportion test, bootstrap intervals, a Bayesian posterior, and
difference-in-differences.

**The value is not the algorithms — those are library calls. It is the refusal to
present a number as more trustworthy than it is.** Three cases, each verified:

- Dropping censored rows makes a payment-timing median optimistic. On the shipped
  mock data: honest 28 days, naive 18. Ten days of working capital, wrong direction.
- A leaked outcome column produces ROC-AUC near 1.0 and a useless model. The tool
  reports that as an **error** naming the culprit, not as a success.
- Observational waiver analysis can invert the sign of the true effect. Verified:
  with the true causal effect positive but waivers targeted at weak borrowers, the
  observational estimate recommends a **0%** waiver. The tool detects the
  confounding and refuses to call the result causal.

A confident wrong number is worse than an honest uncertain one, because someone
acts on it.

## 2. Tech Stack, Libraries & CI/CD

CustomTkinter · pandas · numpy · scipy · scikit-learn · XGBoost · lifelines ·
thefuzz + rapidfuzz · matplotlib · openpyxl + XlsxWriter · PyInstaller.

**Quality gate** (all verified green):

```bash
ruff check app tests main.py mock_data_simulator.py
black --check app tests main.py mock_data_simulator.py
pytest tests/ -q                                    # 84 tests
FINOPS_TEST_FUTURE_STRINGS=1 pytest tests/ -q       # same suite, pandas 3 dtype
python main.py --selftest                           # 10 headless checks
python -m compileall -q app main.py
```

CI jobs: **lint** · **test** (Ubuntu + Windows × py3.11/3.12, installing from
`requirements.txt` so it tests what users get) · **ground-truth** (asserts the
models recover known planted answers) · **packaging-sanity** (AST-scans every
deferred import against `build.spec` hiddenimports) · **forward-compat**
(unpinned latest, `continue-on-error: true`).

ruff deliberately selects `BLE` and `PLC0415`. This codebase makes exactly two
architectural exceptions — broad excepts at UI boundaries, lazy imports inside
functions — and each must carry a `noqa` at the site rather than hide behind a
project-wide ignore.

## 3. Directory Structure

Flat `app/` package. 56 Python files, ~14,200 lines plus 800 in the simulator.

```
main.py                     --selftest / --version / --help before any GUI import
mock_data_simulator.py      Generates test data + GROUND_TRUTH.txt
app/core/                   config · logger · settings · concurrency · session
app/modules/catalog.py      Documentation source of truth (see §4, invariant 4)
app/modules/ingest/         loader · profiler · coerce · mapping
app/modules/prophet/        features · propensity · survival · elasticity
app/modules/sentinel/       matching · anomaly
app/modules/causal/         impact
app/modules/export/         workbook
app/selftest.py             Ten headless checks
app/ui/widgets/             card · how_to_use · feedback · chart_panel ·
                            mapping_panel · data_table · tooltip ·
                            action_button · requirements_card
app/ui/pages/               dashboard · data · prophet · sentinel · causal ·
                            help · settings
tests/                      test_ingest · test_models · test_core_and_sentinel
build.spec  scripts/  installer/  docs/  .github/workflows/ci.yml
```

## 4. Architectural Blueprint

**Invariant 1 — `app/modules/` never imports Tkinter.** Every engine is a plain
library, runnable headless. Machine-checked by both a test and a selftest check.

**Invariant 2 — only the Tk thread touches widgets.** `BackgroundTaskRunner` is a
bounded pool (2 workers: XGBoost and sklearn are already internally
multi-threaded, and stacking fits on a 16 GB laptop causes thrash). Results return
through `UiDispatcher`, a queue drained by Tk `after()`. `TaskContext` carries
cooperative cancellation and progress. Duplicate task ids are refused, which is
what stops a double-clicked Run button fitting the same model twice.

**Invariant 3 — pages and heavy dependencies load lazily.** The registry holds
`"module.path:ClassName"` strings and probes importability at startup, so a broken
`lifelines` disables one nav entry instead of the application. This is *why*
`build.spec` needs explicit `hiddenimports`: PyInstaller's static analyser follows
only module-level imports, and almost nothing here is imported at module level.

**Invariant 4 — documentation is generated, never written twice.**
`app/modules/catalog.py` declares, per module: required and optional column roles,
prerequisites, three workflow steps, a `ButtonDoc` per button (visible explanation
+ hover tooltip), and the statistical caveat. The Help page and every module page
render from it. Renaming a role updates the on-screen requirements automatically.
Documentation kept in a separate help file drifts the moment a role is renamed, and
nobody notices until an analyst follows instructions that no longer match the
dropdowns.

**Invariant 5 — every button explains itself.** `ActionButton` takes a `ButtonDoc`
and renders permanently visible text beneath the button plus a hover tooltip. A
disabled button always states why. A greyed-out control with no explanation is the
most common way a click-driven tool becomes unusable.

**Invariant 6 — matplotlib `pyplot` is never used.** pyplot keeps a global figure
registry; in a long-lived GUI creating a chart per model run, that leaks until the
app is killed. `ChartPanel` owns an explicit `Figure` and closes it on replacement.

## 5. AI-to-AI Debugging Heuristics

**Never test a pandas dtype by comparing against `object`.** pandas 2 stores text
as `object`, pandas 3 as `StringDtype`. Code gating on `dtype != object` silently
stops coercing everything under pandas 3 — amounts stay text, nothing on screen
looks wrong. Always test for the type you are converting **to**
(`is_numeric_dtype`, `is_datetime64_any_dtype`). Carried over from a sibling
project where it shipped; this codebase was written to avoid it and the suite runs
twice to keep it that way.

**A regression test can pass and still prove nothing.** The obvious test for the
above — `astype("string")` on the input — passes against broken code, because
header promotion rebuilds the frame and resets the dtype. Only the global
`future.infer_string` option reproduces it. Before trusting any regression test,
revert the fix and confirm it actually fails.

**Bugs found here by reading test output rather than by crashing** — all four
silent, all four "wrong but plausible":

1. *Profiler stats used `pd.to_numeric`.* A Late Fee column spanning −750 to 1500
   was displayed as "range 0.00 to 0.00", because only the trivially parseable
   values were counted. A wrong range on the mapping screen makes an analyst pick
   the wrong column with confidence. Always use the full parser for display stats.
2. *Feature suggestion excluded columns bound to other roles.* That removed
   outstanding amount and late fee — the most predictive variables available — from
   the model. What must be excluded is different and far more important: anything
   encoding the **outcome**.
3. *The cardinality heuristic was applied to numerics.* A near-unique numeric
   column is *continuous*, not an identifier. Applying an ID test to it silently
   dropped amount, DPD and fee from every model.
4. *Priority bands were multiples of the base rate.* At a 73% repayment rate,
   `rate × 1.5 = 1.10` — above the maximum possible probability — so an entire band
   became unreachable and mid-scoring accounts were labelled "Low yield". Replaced
   with score quantiles. Then a boundary bug: at a 5% base rate the 20th percentile
   clips to exactly 0.0, and `score >= 0.0` matched everything, so the bottom band
   never fired. The lowest boundary must be strict.

**Balance testing on post-treatment columns inverts the conclusion.** The mirror
image of target leakage, and the subtlest bug in this build. `Amount_Recovered` is
derived from the outcome, so it differs between arms *because the campaign worked* —
and the balance check reported that as "the groups were not comparable", turning a
successful campaign into a false confounding alarm. Balance is only meaningful on
characteristics fixed **before** the intervention. `run_impact` now accepts explicit
`covariate_columns`, and when auto-selecting marks columns correlating above 0.55
with the outcome as `likely_post_treatment` and excludes them from the verdict.

**The strongest test for any causal engine is the placebo.** Filter to the
pre-intervention period and re-run. An engine that finds an effect before anything
happened is finding noise. Verified here: p = 0.95 pre-intervention, p = 0.005
post, DiD recovering the planted 8.01% exactly.

**Corner solutions in an optimiser mean the answer is an artefact.** The waiver
simulator's premise is an interior trade-off. An optimum at 0% or 100% means no
peak was found and the recommendation is driven by the assumption or the data range,
not by a balance point. Flagged explicitly. Note that with realistic lending numbers
(principal ≫ fee) interior optima are genuinely rare — itself an honest finding.

**Fuzzy matching must be blocked or it is unusable.** Naive matching is
O(n × m): two 100k files is ten billion comparisons. Three passes, cheapest first
— exact dictionary lookup, then fuzzy restricted to candidates already agreeing on
amount and inside the date window, then amount-and-date requiring a *unique*
candidate so it cannot arbitrarily pair two same-value same-day transactions. A
test asserts comparisons stay under 5% of naive.

**Reference normalisation must not erase the reference.** Stripping channel
prefixes (`UTR`, `NEFT`, `UPI`) turns `utr: neft-000123456789` into `000123456789`
— but a reference that is literally `UPI` would collapse to an empty string and
match every other empty one. Only accept the stripped form if ≥4 characters survive.

**`contamination` sets how many rows are flagged, not how many are wrong.** An
Isolation Forest returns approximately that proportion whatever the data. At 10%,
10% of a clean file is flagged. Documented on screen; default 2%.

**Mock Tk stub limitation.** `_W.__getattr__` returns a no-op callable for any
missing attribute, so `hasattr(page, "_anything")` is always True. Test for the
real type in `__dict__` instead. This cost a debugging cycle: five pages appeared
to fail when only the test was wrong.

**Windows specifics, already handled — keep them.** `.gitattributes` marks `*.bat`
as `-text`, not `eol=crlf`: the latter only applies on checkout, and a ZIP download
never checks out, leaving LF-only batch files that make `cmd.exe` mis-seek on
`goto`. Console output goes through
`stream.reconfigure(encoding="utf-8", errors="replace")` because Windows CI pipes
are cp1252 and a rupee sign otherwise raises. `.gitignore` uses `exports/*`, not
`exports/`, because git cannot re-include a `.gitkeep` beneath an excluded
directory.

**The simulator is a test instrument, not a demo.** `mock_data_simulator.py`
generates from parameters chosen up front and writes them to `GROUND_TRUTH.txt`.
Without knowing the answer in advance, 26 days and 47 days look equally plausible.
The observation window is set close to the true median deliberately (34 days
against a 28-day median, ~41% censored) — a generous window would leave nothing
censored and the handling this data exists to test would never be exercised.
Planted traps with recorded counts: month-end repayment penalty, **targeted**
waivers whose true causal effect is zero, truncated names, missing UTRs, one-digit
typos, duplicate refunds, ghost transactions, bank-only credits, late settlements,
fee-netted amounts.

**All test data is synthetic, deliberately.** This project handles lending data
under a Claude Team plan with no BAA, so no real customer record may enter a prompt
or a fixture. Keep it that way.
