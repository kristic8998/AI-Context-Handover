# LendOps Studio - System Architecture & AI Handover Context

Repo: https://github.com/kristic8998/lendops · Current: **v1.0.0** · Package `lendops`, CLI `lendops` (`--selftest`). CI green on Ubuntu + Windows, Python 3.10/3.12. *(Distinct project from `lendops-toolkit` — same architecture DNA, different modules.)*

## 1. Core Purpose & Business Value

One click-driven desktop app for the three daily jobs of a micro-lending operations team (target demographic: students & young professionals): **Collecta** (delinquency prediction → prioritized, phone-ready calling lists with a plain-English reason per call), **PolicySim** (backtest hypothetical credit rules against the historical book → actual-vs-simulated P&L and default rate, with stated assumptions), **KYC Sentinel** (application fraud screening: shared bank accounts/IDs across different names, underage, age/DOB mismatch, invalid PAN, absurd ask-to-income). Every page: permanent How-to-Use card + "Try with sample data" button; all engines pure and headless-tested (44 tests).

## 2. Tech Stack & Dependencies

customtkinter, pandas, numpy, openpyxl, **scikit-learn** (lazy-imported only inside Collecta's model-upgrade path — engines import without it). Dev: pytest/ruff/black/pyinstaller. Standard CI matrix + headless selftest. Packaging: `LendOps.spec` (collects customtkinter assets, hiddenimport `sklearn.linear_model`, bundles `sample_data/`), `scripts/build_windows.bat` + `build_portable.bat`, `installer/lendops.iss`.

## 3. Complete Directory Structure

```
lendops/
├── src/lendops/
│   ├── __init__.py  __main__.py  app.py  selftest.py
│   ├── core/        # paths.py (%LOCALAPPDATA%/LendOps, LENDOPS_HOME), config.py (JSON),
│   │                # tasks.py (TaskRunner), demo.py (seeded active/historical/applications data)
│   ├── modules/     # PURE engines: tabular.py, collecta.py, policysim.py, kyc.py
│   └── ui/          # widgets.py (HelperCard, KpiCard, DataGrid w/ risk tinting, Toast,
│                    #   run_in_thread), pages/ (home, collecta, policysim, kyc)
├── tests/  sample_data/ (active_loans.csv, historical_loans.csv, daily_applications.csv)
├── scripts/  installer/  LendOps.spec  launcher.py  docs/ (INSTALL, USER_GUIDE, TROUBLESHOOTING)
```

## 4. Architectural Blueprint

### 4.1 Collecta (`modules/collecta.py`)
Transparent weighted scoring over hint-detected factors, `WEIGHTS = dpd .40 / missed .25 / burden(EMI÷income ÷0.6) .20 / util(outstanding÷amount) .10 / student .05`, each factor clipped to [0,1], score = 100×Σ. **Severe-delinquency floor**: dpd ≥ 60 ⇒ score ≥ 75 regardless of other columns (a real bug fix — a 90-DPD loan must never be Medium). Bands: ≥60 High, ≥35 Medium. **Automatic model upgrade**: if the file has an outcome column (`defaulted/npa/...` hints), both classes present, and ≥30 rows → `LogisticRegression` trained *on that file*, `predict_proba×100` replaces the heuristic score; `summary.model` tells the UI which path ran. Explainability: `top_risk_driver` = argmax of weighted contributions mapped to human text ("already past due", "EMI is heavy vs income"…). Export: Calling List (High ± Medium, sorted) / All Loans Scored / Summary.

### 4.2 PolicySim (`modules/policysim.py`)
Requires an outcome column (friendly error otherwise). `PolicyRules(max_loan_amount, min_monthly_income, max_loan_to_income, exclude_students, interest_rate_pct)` — `None` = rule off. Rules applied in order; **first failing rule becomes `decline_reason`**. Economics (stated in `result.assumptions` and on-screen): flat interest = amount × APR × tenure/12; defaulters earn `default_interest_fraction` (0.5) of scheduled interest and lose `loss_given_default` (0.65) of principal; missing rate column ⇒ 24% APR assumed (noted). `SimulationResult` carries actual vs simulated `PortfolioMetrics` (loans, disbursed, interest, losses, net profit, default rate) + approval rate + annotated frame. It is a **directional what-if tool, not accounting** — keep that honesty in UI and docs.

### 4.3 KYC Sentinel (`modules/kyc.py`)
Frame is `reset_index(drop=True)` first (all flag bookkeeping is positional). Cross-row: `flag_shared(column, check, alert_on_name_clash)` — shared bank account / ID with **different names ⇒ alert** (mule pattern), same name ⇒ watch (resubmission); shared phone/email ⇒ watch. Per-row: underage (<18 from DOB, `pd.to_datetime(dayfirst=True)`, or stated age), age-vs-DOB mismatch >1.5y ⇒ alert; PAN regex `^[A-Z]{5}[0-9]{4}[A-Z]$` (only when the id column name contains "pan") ⇒ watch; missing critical field ⇒ watch; requested > 20× income ⇒ watch. Output frame gains `flags` (semicolon reasons), `flag_count`, `severity`, alerts sorted first; `check_counts` powers the Summary sheet.

### 4.4 UI specifics
PolicySim page uses `_RuleSlider` (checkbox-gated CTkSlider + live value label; unticked ⇒ `value() is None` ⇒ rule off) and renders the comparison as a label grid (not KPI-card spam) + a declined-loans DataGrid. Collecta/KYC grids tint rows by `risk_band`/`severity` via DataGrid's `tint_by=` (tag map in `widgets._TINT_TAGS`). Sample-data buttons call `core.demo` generators directly in-memory (no temp files) — saving a Builder-style recipe against sample data is intentionally refused where a real path is required.

## 5. AI-to-AI Debugging Heuristics

1. **"Model didn't kick in"**: expected unless outcome column exists with both classes and n ≥ 30 — that's the guard, not a bug. Check `result.summary.model` string.
2. **Everything scores Low on a sparse file**: only the columns that exist contribute; the dpd ≥ 60 floor still guarantees High for severe delinquency. Never remove `num_series`'s fillna(0) behaviour — sparse files are a supported input.
3. **sklearn ImportError headless/CI**: something imported sklearn at module top. It must stay inside `analyze()`'s training branch (lazy) so engines import without it.
4. **PolicySim profit looks "wrong"**: read `assumptions` first — flat interest + LGD + default-interest-fraction are simplifications; a rule set that lowers default rate can legitimately lower net profit too (it declines good loans). That trade-off display is the product.
5. **KYC flags on wrong rows** after any refactor: positional bookkeeping requires the initial `reset_index(drop=True)`; iterate with positions, not original index labels.
6. **DOB parsing warnings/misreads**: inputs are dayfirst Indian formats; keep `dayfirst=True`, coerce errors, and treat unparseable DOB as "no DOB" (falls back to stated age), never as underage.
7. Threading/encoding/version-bump/packaging heuristics: identical to the cross-project conventions in README.md (TaskRunner + after(0); `_utf8_console`; bump pyproject/__init__/iss/bat together; PyInstaller hiddenimports).

## 6. Mock Data Simulation & Execution Guide

1. `python mock_data_simulator.py` — repo root; writes `mock_data/active_loans.xlsx`, `active_loans_sparse.csv` (3 columns only), `historical_loans.xlsx` (outcome column, both classes), `daily_applications.xlsx`. Seeded and reproducible.
2. `lendops` (or `python -m lendops`) → upload each file on its page.
3. Expected results (verify, don't admire): **Collecta** — row `LN50000` (dpd 90, otherwise rosy) must score **High** (severe-DPD floor); the sparse CSV must still band all rows. **PolicySim** — sliders move approval/default/profit; row `HL90000` has a NaN rate (24% APR fallback path). **KYC Sentinel** — exactly **3 alerts** planted (shared bank account under two different names ×2 rows, underage DOB) plus watch-level: invalid PAN, 25× income ask, shared phone (2 rows), missing bank account.
4. Headless validation (2026-07-26): `collecta.analyze`, `policysim.simulate`, `kyc.scan` run green on these exact files; KYC severity counts came back `{alert: 3, watch: 7}`.
