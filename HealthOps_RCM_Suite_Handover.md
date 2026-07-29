# HealthOps RCM Suite - System Architecture & AI Handover Context

Repo: https://github.com/kristic8998/healthops-rcm-suite · Current: **v1.0.0** · Package `healthops`, CLI `healthops` (`--selftest`). CI green (run #1) on Ubuntu + Windows, Python 3.10/3.12. First-hand: written by the architect who built it. **Every byte of data in this repo is synthetic — no PHI exists or is needed anywhere.**

## 1. Core Purpose & Business Value

Six US-healthcare revenue-cycle tools in one click-driven desktop app for a claims analyst: **EDI Parser Pro** (X12 837P/837I/835/Rx with the 835 CAS split of Member PR vs Provider CO responsibility and Rx void detection), **ClaimGuard & ORS Router** (declarative rule engine routing claims to operational queues), **DenialPredict AI** (explained denial-risk scores), **AccuSync Reconciler** (under/overpayment detection + OOPM/deductible accumulators), **Gov-Comply Auditor** (Medicare TFL + MBI format + demo LCD/NCD), **AR Cashflow Forecaster** (aging buckets → expected-cash curve). Every page: permanent 3-step card + synthetic sample button; engines are pure and headless-tested (35 tests + 12 selftest checks).

## 2. Tech Stack & Dependencies

customtkinter, pandas, numpy, openpyxl, PyYAML. **Optional `[ml]` extra**: lightgbm, scikit-learn, shap — DenialPredict lazy-imports them and degrades honestly without them. Dev: pytest/ruff/black/pyinstaller. Standard house CI + packaging (`HealthOps.spec`, `scripts/build_windows.bat` + `build_portable.bat`, `installer/healthops.iss`); spec `excludes` matplotlib/scipy and hiddenimports `yaml`.

## 3. Complete Directory Structure

```
healthops-rcm-suite/
├── src/healthops/
│   ├── __init__.py  __main__.py  app.py  selftest.py
│   ├── core/        # paths (%LOCALAPPDATA%/HealthOps), config.py (JSON),
│   │                # tasks.py (TaskRunner), demo.py (in-package synthetic samples)
│   ├── modules/     # PURE engines: tabular.py (find_col/num/text/parse_amount),
│   │                # edi.py, rules.py, denial_ai.py, reconcile.py, govcomply.py, ar_forecast.py
│   └── ui/          # widgets.py (house set), pages/ (home + 6 module pages)
├── tests/  mock_data_simulator.py (writes GROUND_TRUTH.txt)
├── scripts/  installer/  HealthOps.spec  launcher.py  docs/  .github/workflows/ci.yml
```

## 4. Architectural Blueprint

### 4.1 EDI (`modules/edi.py`) — Factory Pattern
`ParserFactory.for_transaction(ST-split-segments)` → `Parser837P` (SV1/CPT) / `Parser837I` (SV2/rev; detected by SV2 presence or 005010X223) / `Parser835` / `ParserRx` (LIN/NDC). All produce uniform `ParsedClaim`. **CAS split**: repeating (reason, amount, qty) triplets; group `PR` → member (reason 1=deductible/2=coinsurance/3=copay into `member_breakdown`), `CO/OA/PI` → provider. Claim frequency code in CLM05-3 = `7|8` ⇒ `is_void`. Malformed segments are logged + skipped and surface as claim notes — a bad remit line must never kill a batch.

### 4.2 Rules/ORS (`modules/rules.py`) — Strategy Pattern
`STRATEGIES: dict[str, (frame, params) -> bool Series]` (hmo_requires_auth, oon_high_dollar, network_penalty, payer_equals, field_equals, amount_at_least). Rules come from JSON/YAML (`load_rules`), evaluate in order, vectorised; every match appends its flag, **first queue match wins** (firewall ordering — specific rules on top); unmatched → default queue. New strategy = one registry entry.

### 4.3 DenialPredict (`modules/denial_ai.py`) — honest model ladder
Features are pure pandas in [0,1] (oon, hmo_no_auth, high_dollar, medicare, denied_before, rx_void). Ladder: LightGBM → XGBoost → sklearn logistic → transparent WEIGHTS floor; training only when an outcome column with both classes and n≥40 exists. Explanations: SHAP TreeExplainer when importable, else feature-importance/coefficient contributions, else rule contributions — `summary.model`/`summary.explainer` always state which ran. Every row gets a plain-English `top_driver`.

### 4.4 AccuSync (`modules/reconcile.py`)
`paid - allowed` with ±0.01 tolerance → UNDERPAYMENT / OVERPAYMENT / PAID_CORRECT via one `np.select`. Accumulators: DEDUCTIBLE/OOPM `_MET|_EXCEEDED|_NEAR` (90% early warning); `MEMBER_REFUND_DUE` when member paid despite OOPM exceeded.

### 4.5 Gov-Comply (`modules/govcomply.py`)
Only Medicare/Medicaid rows are audited. TFL: age > 365d ⇒ `TFL_BREACH`, last-60-days ⇒ `TFL_RISK` (audit_date injectable for determinism). `validate_mbi` implements the CMS character classes (`[1-9]` pos1, letters exclude S/L/O/I/B/Z, positions 3/6 alphanumeric) and names the failing position. `DUMMY_LCD_NCD` is a clearly-marked demo table — real coverage policy is payer data, not code.

### 4.6 AR Forecast (`modules/ar_forecast.py`)
`pd.cut` on fixed edges (30/60/90/120/∞) → per-bucket collection probability + lag-days (visible dict defaults, overridable), denied haircut ×0.60 → per-claim Expected Cash + landing date → weekly `Grouper(freq="W-MON")` curve. `assumptions` list ships with every result — directional, not accounting.

### 4.7 House contracts
Engines never import Tkinter; UI work through `TaskRunner.submit` + `run_in_thread` (`widget.after(0, …)`); `find_col` hints everywhere; all money columns through `parse_amount_series` (text money, accounting negatives, Cr/Dr, percents); friendly ValueErrors → toasts.

## 5. AI-to-AI Debugging Heuristics

1. **835 member/provider totals look wrong**: check CAS triplet stride first — the parser walks (reason, amount, qty) in threes; a payer omitting qty mid-segment shifts the frame. Compare against the CLP05 patient-resp note the parser records.
2. **A claim landed in the wrong queue**: rule ORDER decides — first queue match wins. Specific rules (high-dollar OON) must precede broad ones (Medicare). Fix the JSON ordering, not the engine.
3. **"Model didn't train"**: expected unless an outcome column with BOTH classes and ≥40 rows exists — read `summary.model`. SHAP failing (numba/version issues) degrades to feature importances by design; never let SHAP exceptions kill scoring.
4. **MBI false negatives**: remember positions 3 and 6 are alphanumeric while 2/5/8/9 are letters-only; S/L/O/I/B/Z are illegal everywhere. The validator's reason string names the failing position — trust it.
5. **TFL flags shift day to day**: audit_date defaults to today; tests and the selftest pin `pd.Timestamp("2026-07-29")`. Pass audit_date explicitly in anything deterministic.
6. **Forecast weekly curve ≠ per-claim sum**: someone resampled after rounding. Round per-claim, then group — the selftest asserts conservation to the cent.
7. Engines must stay importable without lightgbm/shap/yaml installed (lazy imports); adding a top-level heavy import breaks the CLI-on-clean-machine story and the frozen exe.
8. House rules apply: cp1252 selftest shim, 4-file version bump (pyproject/`__init__`/iss/build_portable.bat), verify the GitHub tree after web uploads, CI workflow committed last.

## 6. Mock Data Simulation & Execution Guide

1. `python mock_data_simulator.py` — repo root (pandas/numpy/openpyxl). Writes `mock_data/`: `mock_837_835_EDI.txt`, `mock_claims_master.xlsx`, `mock_payment_reconciliation.xlsx` and **`GROUND_TRUTH.txt`** (the exact expected outcomes — verify, don't eyeball). Seeded `20260729`.
2. `healthops` (or `python -m healthops`) → per page: Parser ← EDI txt; Router/Denial/Comply/Forecast ← claims master; AccuSync ← payment reconciliation.
3. Expected (must match GROUND_TRUTH to the cent): 835 → PR 40.00 (copay 20 + deductible 20) vs CO 100.00 and the malformed CLP skipped with a note; `CLM100000` → AUTH_MISSING → HMO Auth Review Queue; `CLM100001` → High-Dollar OON Queue; `CLM100005` → top risk band with driver text; recon `CLM100000` UNDERPAYMENT 120.00 / `CLM100001` OVERPAYMENT 500.00 / DEDUCTIBLE_MET / OOPM_EXCEEDED rows; `CLM100002` MBI invalid at position 2; `CLM100003` TFL_BREACH; forecast weekly curve sums exactly to per-claim expected cash.
4. Headless equivalent: `healthops --selftest` runs the same 12 ground-truth checks with no Tkinter (CI does this on every push).
