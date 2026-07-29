# FinOps Predictive Suite — Handover

**Repository:** https://github.com/kristic8998/finops-predictive-suite (private)
**Version:** 1.0.0 · **Date:** 29 July 2026 · **CI:** green (run #3, 3m 57s)
**Scale:** 76 files, ~18,300 lines Python, 120 tests, 12 self-test checks, 14 ground-truth checks

---

## 1. What This Project Is

A Windows desktop application (CustomTkinter) for lending operations. Four modules, each
aimed at one expensive problem, each built to state plainly when its own answer should not
be trusted. Everything runs locally — no network calls, no uploads, no accounts.

| Module | Problem | Method |
|---|---|---|
| **DropGuard AI** | Applicants abandoning the journey | Distinct-user funnel + absorbing Markov chain (`N=(I−Q)⁻¹`, `B=N·R`) |
| **NudgeMatrix** | Collections effort landing badly | Beta-posterior bandit; rollout on lower credible bound |
| **RiskRadar** | Defaults arriving without warning | LightGBM→XGBoost→RandomForest + leakage refusal + fair-lending proxy audit |
| **PulseNLP** | Nobody can read 10,000 reviews | Lending-tuned sentiment + LDA/NMF topics + deterministic keyword groups |

**Design invariant:** `app/modules/` never imports Tkinter. Machine-enforced by a test and
by the self-test, so analysis code stays headlessly testable and thread-safe.

---

## 2. Architecture

```
main.py                      Entry point: --selftest, --make-mock-data, --version
mock_data_simulator.py       Generates 4 datasets + GROUND_TRUTH.txt
run.bat                      One-click launcher (venv, deps, mock data, launch)

app/core/                    config, logger, settings (atomic JSON), session, concurrency
app/modules/                 ALL analysis — zero UI imports
  catalog.py                 Single source of truth for every on-screen explanation
  ingest/                    loader, profiler, coerce, mapping
  dropguard/                 funnel.py, markov.py
  nudgematrix/               bandit.py
  riskradar/                 scoring.py
  pulsenlp/                  sentiment.py, topics.py
  export/                    workbook.py
app/ui/
  registry.py                Pages declared as "module:Class" strings, imported lazily
  pages/                     8 screens + base_page
  widgets/                   action_button, chart_panel, mapping_panel, requirements_card…
app/selftest.py              12 headless diagnostic checks

tests/                       120 tests, run twice (pandas 2 + pandas 3 string semantics)
tools/verify_ground_truth.py 14 checks that the engines still recover every planted fault
build.spec                   PyInstaller — hiddenimports cross-checked by CI
installer/finops_suite.iss   Per-user Inno Setup, no admin rights
scripts/build_windows.bat    Lint→test→selftest→package→**selftest the frozen exe**
```

**Threading contract:** `UiDispatcher` (thread-safe queue drained by Tk `after()`) +
`BackgroundTaskRunner` (bounded pool, max_workers=2) + `TaskContext` (cooperative cancel,
progress). Duplicate task ids are refused, which is what stops a double-clicked Run button
fitting the same model twice.

---

## 3. Key Decisions (all arrived at by measurement)

**Sentiment needed a lending vocabulary overlay.** Raw VADER is social-media English.
Measured against vaderSentiment 3.3: `crashes`, `buggy`, `hidden`, `deducted`, `pending`,
`declined` are **absent entirely**, while `please` (+1.3), `approved` (+1.8) and `perfectly`
(+3.2) score as praise. "APP CRASHES EVERY TIME, PLEASE FIX" scored **+0.32**.

**Sentiment also needed clause-level scoring.** VADER's negation is a blind 3-token window
with no syntax awareness. "aadhaar scan not working, verification stuck…" scores −0.22 and
−0.49 as separate clauses but **+0.19** together, because the `not` from "not working" lands
within three tokens of `stuck` and flips it from −2.2 to +1.6. Four combining rules were
measured on 2,400 rated reviews; **strongest-clause-by-magnitude** won on every metric:

| Variant | Balanced acc. | Agreement w/ stars | 1–3★ called positive |
|---|---|---|---|
| Raw VADER | 79.5% | 71.5% | 581 |
| + lending lexicon | 96.3% | 94.1% | 133 |
| + clause mean | 93.4% | 95.5% | 95 |
| + clause min | 87.9% | 93.4% | 76 (*+64 false negatives*) |
| **+ strongest clause** | **98.0%** | **96.6%** | **76** |

**LDA is the wrong default on review text.** Median document is 8 usable words — too little
evidence for its Dirichlet posterior to sharpen. Against known planted themes: LDA ARI 0.26
(49% purity), NMF ARI 0.55 (71%). Same data, vocabulary and seed. So `algorithm="auto"` fits
both and keeps the higher coherence (coherence was validated as a selector first — it agreed
with ARI at 5, 6 and 7 topics).

**Keyword groups sit alongside clustering, not instead of it.** Clustering draws its own
boundaries: the planted 23% OCR failure emerged as three clusters of 13%/9%/8%, each looking
minor. Cosine similarity cannot merge them — NMF's `nndsvd` init makes components
near-orthogonal (all off-diagonals 0.00–0.19). Keyword grouping measured it at **22.4%**
against a planted 23.4%. Clustering discovers; keyword groups measure and track.

**The bandit exploits on the floor, not the point estimate.** The first version ranked by
Thompson sampling's P(best) and recommended a 5-attempt arm over a 4,000-attempt arm. P(best)
rewards uncertainty — right for deciding what to *test*, wrong for what to *deploy*. Rollout
uses the lower credible bound plus a minimum-evidence gate; exploration is planned separately
and its cost priced. The starved arm still shows high P(best) (that is correct) but cannot
drive rollout.

**A very high AUC is an error, not a success.** Nothing known at underwriting predicts
default at 0.99. `Collections_Notes` (populated only for defaulted loans) drives AUC to
1.0000 → verdict `error`, culprit named. Clean model: 0.7372, verdict `ok`.

**Proxy audit separates hidden proxies from direct use.** Earlier versions buried the real
finding twice: first self-matches (Age↔age at 100%) crowded out `Device_Type`, then sensitive
attributes appeared as proxies for each other. Now `Device_Type` is found at 60% via
correlation ratio, listed as a hidden proxy, with direct use (Age, City, Monthly_Income)
reported separately as a policy question.

**A mistyped column name is an error, not a fallback.** `require_columns()` was added after
`time_column="Timestamp"` on a file whose column is `Event_Timestamp` was silently ignored —
the funnel fell back to appearance order in a shuffled event log and reported the wrong
bottleneck (App Open 27% instead of KYC Started 54%) with complete confidence.

---

## 4. Bugs Found and Fixed

| Bug | Symptom | Fix |
|---|---|---|
| VADER lexicon gap | 581/2,067 complaints labelled positive | 100-term lending overlay |
| VADER negation leak across clauses | Two negative clauses → positive document | Clause-level, strongest-by-magnitude |
| LDA/NMF mis-selection | Themes blurred, ARI 0.26 | Fit both, keep higher coherence |
| `assignments` 0-based vs `Topic.number` 1-based | Export joins off by one | Made assignments 1-based |
| `risk_band()` returned long action text | **0 high-risk loans on a 19%-default book** | One `RISK_BANDS` table + `risk_band_name()` |
| Silent column-name fallback | Wrong bottleneck, no warning | `require_columns()` in every engine |
| `groups={}` fell back to defaults | Measured a different issue set than asked | `is None` check, explicit error |
| Duplicate feature in proxy verdict | `Bureau_Score` named twice | Dedupe + single-finding phrasing |
| `Days_Past_Due` not flagged | Delinquency leak undetected | 15 delinquency hints added |
| Markov headline vacuous | "P(Disbursed→Disbursed)=100%" | `transient_absorption`, `real_retries` |
| Profiler used `pd.to_numeric` | "Rs 1,20,000" range shown as 0.00–0.00 | Use the shared parser |
| pandas 3 `dtype != object` | All coercion skipped | Test for target type; AST-enforced by test |

---

## 5. Required Mock Files

Generated by `python mock_data_simulator.py` (or Start Menu → *Generate mock test data*):

| File | Rows | Planted fault |
|---|---|---|
| `mock_funnel_data.xlsx` | 23,367 | KYC Started 54% bottleneck; 8.4% retries so row counts overstate reach |
| `mock_collection_history.xlsx` | — | True best = WhatsApp @ 19:00; trap = Email @ 17:00, 60% on 25 attempts |
| `mock_underwriting_data.xlsx` | 5,000 | `Device_Type` proxies income; `Collections_Notes` is a post-outcome leak |
| `mock_customer_reviews.xlsx` | 2,400 | 23.4% KYC-OCR bug in 5 phrasings; 80 Hinglish, 15 emoji-only, 44 empty |
| `GROUND_TRUTH.txt` | — | The correct answers for all four |

---

## 6. Mock Data Simulation & Execution Guide

**Run this once, before opening the app.**

```bat
python mock_data_simulator.py
```

Frozen build: Start Menu → **Generate mock test data**, or
`FinOpsPredictiveSuite.exe --make-mock-data`.

It writes four Excel files plus **GROUND_TRUTH.txt** into `mock_data/`.

**GROUND_TRUTH.txt is the point of the whole exercise.** It states the correct answer for
every dataset — which funnel step is genuinely the bottleneck, which channel genuinely
performs best, exactly where the hidden bug is and how big. Run a module, read the file, and
you can tell whether the app is **right** rather than merely plausible.

The planted faults are adversarial by design: a bottleneck a row-count would misidentify, a
channel whose flattering rate comes from 25 attempts, a predictor that stands in for income,
a leak that produces a perfect-looking model, an OCR failure phrased five ways so no single
keyword finds all of it, and 95 rows an English sentiment model cannot read at all.

**Verify the promise still holds:**

```bat
python -m tools.verify_ground_truth
```

14 checks across all four engines. Runs in CI on every commit, so GROUND_TRUTH.txt cannot
quietly become a lie. Current status: **14/14 pass.**

Selected recoveries: funnel bottleneck = KYC Started ✓ · Markov absorption 15.81% vs funnel
15.82% ✓ · bandit picks WhatsApp ✓ · clean AUC 0.7372 / leaked AUC 1.0000 → error ✓ ·
Device_Type proxy found ✓ · negative share 82.8% vs planted 86% ✓ · Hinglish 80/80 ✓ ·
emoji 15/15 ✓ · KYC issue 22.4% vs planted 23.4% ✓

**Diagnose a machine:**

```bat
python main.py --selftest
```

12 checks: Python version, required/optional libraries, analysis code is UI-free, all 8 pages
import, documentation catalog complete, risk band table consistent, sentiment lexicon
non-contradictory, settings round-trip, folders writable, and all four engines end-to-end on
synthetic in-memory data. Exit code = number of failures. On a Python without Tkinter the
page check reports WARN, not FAIL, so CI stays honest rather than permanently red.

---

## 7. Current State & Next Steps

**Done:** all four modules, all 8 screens, in-app documentation generated from `catalog.py`,
120 tests green under both pandas modes, lint and format clean, self-test green, ground truth
verified, CI green on Windows + Linux across Python 3.10–3.12, PyInstaller spec and per-user
installer written and CI-validated.

**Not yet done:**

- No `assets/app.ico` — `build.spec` treats it as optional so the build works, but the exe
  ships with the default PyInstaller icon.
- `scripts\build_windows.bat` has not been run on real Windows hardware. It is the only step
  that exercises PyInstaller and the frozen self-test.
- `suggest_topic_count()` is implemented and tested but not yet wired to a UI control.
