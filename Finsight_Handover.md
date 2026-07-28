# FinSight - System Architecture & AI Handover Context

Repo: https://github.com/kristic8998/finsight · Current: **v1.6.0** (use this tag; **v1.5.0's tag is broken** — see §6) · CI: GitHub Actions, green on Ubuntu + Windows, Python 3.10/3.12.

## 1. Core Purpose & Business Value

FinSight is a Windows desktop suite that replaces the pile of Excel sheets, SQL scripts and manual MIS work inside an NBFC/FinTech lending team. One offline-first app: executive KPIs and health scoring, English-question NLQ over the loan book, SQL studio, Excel tools, 3-file reconciliation with root-cause analysis, MIS pack generation, explainable analytics (forecast/anomalies/segments/risk), a Data Quality Center, an API Explorer, a drop-in plugin system, and — since v1.5 — **MIS Studio**: a zero-code Visual MIS Builder, three One-Click Lending Templates, and a Visual Auto-Reporter (scheduling). It ships with a deterministic synthetic lending book so everything works on first launch; production data connects via SQLAlchemy (SQLite/MSSQL/Azure SQL). Business value: a senior data analyst hands non-technical colleagues a double-click install that does their daily reporting with zero training and zero cloud exposure of lending data.

## 2. Tech Stack & Dependencies

- Python 3.10–3.12, **CustomTkinter** UI (+ ttk.Treeview grids), **pandas/numpy**, **matplotlib** (embedded via `FigureCanvasTkAgg` — Analytics page and MIS Builder chart), **scikit-learn** (explainable models: KMeans, logistic; also Collecta-style scoring in the DQ engine tests), **SQLAlchemy** (+ optional `pyodbc` extra for MSSQL), **pydantic + PyYAML** (typed config), **openpyxl** (all Excel writing), **keyring** (credentials → Windows Credential Manager), **requests** (API Explorer).
- Dev: pytest (155 tests), ruff (with `known-first-party = ["finsight"]` pinned — do not remove), black (line-length 100), PyInstaller.
- CI (`.github/workflows/ci.yml`): matrix ubuntu/windows × 3.10/3.12 → install `.[dev]` → ruff → black --check → pytest → `finsight --selftest` (headless, 20 subsystems) → `compileall src`. The selftest is the real end-to-end gate; keep new subsystems registered in `selftest.py`.
- Packaging: `FinSight.spec` (one-folder), `scripts/build_windows.bat` (venv + selftest preflight + freeze), `scripts/build_portable.bat` (zip), `installer/finsight.iss` (Inno Setup, per-user, fixed AppId `{{7F3C2A54-9B1E-4D2A-9C7E-FIN51GHT2026}}` enables in-place upgrades). Exe is unsigned → SmartScreen prompt is expected/documented.

## 3. Complete Directory Structure

```
finsight/
├── src/finsight/
│   ├── __init__.py            # APP_NAME, __version__ (keep in sync with pyproject/iss/bat!)
│   ├── app.py                 # shell: _PAGES nav table, _resolve_navigation (merges plugins),
│   │                          #   FinSightApp(CTk), create_app() composition root, main()
│   ├── selftest.py            # `finsight --selftest`, 20 checks, UTF-8 console shim
│   ├── core/                  # config.py (pydantic+YAML), paths.py (%LOCALAPPDATA%/FinSight,
│   │                          #   FINSIGHT_HOME override), appdb.py (SQLite app state),
│   │                          #   registry.py (modules/actions/plugins for palette+sidebar),
│   │                          #   tasks.py (TaskRunner + retry), backup.py, credentials.py,
│   │                          #   logging_setup.py, plugins.py (drop-in discovery)
│   ├── data/                  # connections.py (ConnectionManager, QueryResult, row caps),
│   │                          #   demo_data.py (seeded synthetic lending DB),
│   │                          #   queries.py (LendingDataService — ALL KPI math lives here)
│   ├── modules/               # PURE services, no UI imports:
│   │   ├── executive.py  analytics.py  nlq.py  recon.py  investigate.py  excel_tools.py
│   │   ├── mis.py  mis_catalog.py  sql_studio.py  automation.py  productivity.py
│   │   ├── data_quality.py    # streaming profiler (chunked == whole, test-asserted)
│   │   ├── api_explorer.py    # requests wrapper, injectable session
│   │   ├── mis_builder.py     # v1.5: BuilderConfig/build_pivot/export_pivot + SavedReport JSON store
│   │   ├── mis_templates.py   # v1.5: 3 lending templates, TemplateError, TEMPLATES registry
│   │   ├── auto_reporter.py   # v1.5: ReportJob, compute_next_run, AutoReporter daemon
│   │   ├── excel_style.py     # v1.5: shared write_formatted_sheet (house Excel style)
│   │   └── mis_samples.py     # v1.5: sample_lending_dataset(n, seed) for demos/tests
│   ├── plugins/               # built-in drop-ins (example_toolkit.py); user dir:
│   │                          #   %LOCALAPPDATA%/FinSight/plugins/
│   └── ui/
│       ├── context.py         # AppContext dataclass — hand-wired DI container, use_connection()
│       ├── widgets.py         # ACCENT/GOOD/WATCH/ALERT, KpiCard, Section, HelperCard,
│       │                      #   FriendlyDialog/show_friendly_error, DataGrid (page-capped,
│       │                      #   selected_index()), Toast, style_treeview, run_in_thread
│       ├── palette.py         # Ctrl+K command palette (reads core.registry)
│       └── pages/             # one file per sidebar page + PAGE_FACTORIES dict in __init__.py
│           ├── executive_page.py ask_page.py sql_page.py excel_page.py dq_page.py
│           ├── recon_page.py mis_page.py analytics_page.py automation_page.py api_page.py
│           ├── productivity_page.py settings_page.py
│           ├── mis_studio_page.py     # v1.5 host page: CTkTabview with 3 tabs
│           ├── mis_builder_tab.py mis_templates_tab.py auto_reporter_tab.py
├── tests/                     # 155 tests; per-engine files (test_mis_builder/_templates/_auto_reporter…)
├── sample_data/  docs/ (INSTALL, USER_GUIDE, TROUBLESHOOTING, USER_MANUAL, ARCHITECTURE,
│                        DEVELOPER_GUIDE, PLUGINS)  scripts/  installer/  FinSight.spec
```

## 4. Architectural Blueprint

### 4.1 Composition & navigation
`app.create_app()` → `load_config()` → `generate_demo_db()` → `ui.context.build_context(config)` wires **every** service into an `AppContext` dataclass (registry, TaskRunner(4), ConnectionManager, LendingDataService, executive/analytics/nlq/mis/sql/automation/productivity, `extras: dict`). `AppContext.use_connection(name)` re-points all data-driven services at another DB — mutate services through this method only. The shell's `_PAGES` table (id, title, glyph, order) is merged with discovered plugins in `_resolve_navigation()`; pages are lazily constructed from `PAGE_FACTORIES[page_id](host, app)` and cached. Pages read services via `self.app.context`; optional `on_show()` hook fires on navigation.

### 4.2 Threading model (critical invariant)
`core.tasks.TaskRunner` wraps a `ThreadPoolExecutor(4)`; callbacks fire **on the worker thread**, so every UI page routes through `widgets.run_in_thread`, which wraps callbacks in `widget.after(0, …)`. Two long-lived background loops exist: `modules.automation.AutomationCenter` (poll thread for scheduled jobs/folder watch) and `modules.auto_reporter.AutoReporter` (20 s daemon loop). Both are stopped in `FinSightApp._on_close` — AutoReporter lives in `context.extras["auto_reporter"]` (created lazily by the Auto-Reporter tab; singleton, `start()` idempotent, `_on_event` rebound to the live tab on rebuild).

### 4.3 Data layer & schemas
- **App state** (`core/appdb.py`): SQLite at `%LOCALAPPDATA%/FinSight/finsight.db` — saved connections, SQL history/library, notes/kanban, MIS templates, automation runs.
- **Canonical lending schema** consumed by executive/NLQ/analytics: `branches(id,name,…) officers loans(id,branch_id,customer_name,product,principal,status[active|closed|npa],…) payments(loan_id,due_date,amount_due,amount_paid,paid_date) collection_targets`. SQL Studio/Excel/Recon work on arbitrary schemas; the analytics stack needs views shaped like this.
- **JSON stores (v1.5)**: `mis_builder_reports.json` (list of `{name, source_path, config{group_by,value,aggregate,split_by}}`) and `auto_reports.json` (list of `ReportJob` dataclass dicts). Both loaders swallow corrupt files with a warning — startup must never fail on bad stores.

### 4.4 MIS Studio (v1.5) engine contracts
- `mis_builder.build_pivot(df, BuilderConfig)`: validates columns (friendly errors), `groupby`/`pivot_table` on `AGGREGATES = {Sum,Average,Count,Minimum,Maximum}`, sorts desc, appends a **TOTAL row** (sum/count → grand total; mean/min/max → recomputed overall). `export_pivot` → `excel_style.write_formatted_sheet` (title, filled header, widths, `#,##0.00`, bold TOTAL, freeze panes).
- `mis_templates.run_template(key, df)` for keys `daily_disbursement | collection_dpd | portfolio_health` → `TemplateResult(kpis, sheets, notes)`. Missing required columns raise `TemplateError` (user-facing text). DPD buckets via `pd.cut` edges `(-1,0,30,60,90,inf)` → Current/1-30/31-60/61-90/90+; PAR-n = outstanding where dpd>n / total; collection efficiency only if due&paid columns found (else a note, not a failure).
- `auto_reporter.compute_next_run(job, now)` is **pure and unit-tested** (daily; weekly by weekday; monthly with short-month clamp via `calendar.monthrange`, year rollover). `AutoReporter.run_due_jobs(now)` is the deterministic firing core (the thread just calls it); jobs reschedule **after failures too** (never stuck). `_generate()` dispatches `kind = "template:<key>" | "builder:<saved name>"`. Honest scope surfaced in UI: jobs run while the app is open.

### 4.5 Other engines in one line each
`recon.reconcile` (key+amount tolerance matching, dup detection) + `investigate.py` (root-cause decomposition, typo pairs via difflib); `data_quality` streaming accumulator (distinct/reservoir/dup-hash caps `1e5/2e5/3e6`) — chunked and whole must produce identical reports (tested); `api_explorer.ApiExplorer(session=…)` fake-transport-testable; `core/plugins.py` discovers `FinSightPlugin` subclasses (pkgutil for frozen builds, filesystem for user dir), broken plugins are logged and skipped, ids validated, built-ins win collisions.

**v1.6.0 additions:** `modules/amounts.py` is the ONE money parser (text amounts, accounting negatives, Cr/Dr, percents) — builder + all templates route through `parse_amount_series`; Builder gained **Median** and **Count Distinct** aggregates whose TOTAL rows recompute over the whole dataset; fully blank rows are dropped before pivoting.

## 5. AI-to-AI Debugging Heuristics (READ BEFORE TOUCHING CODE)

1. **Tk threading errors** (`RuntimeError: main thread is not in main loop`, frozen UI, ghost widgets): some code path is touching a widget from a worker. Fix by routing through `run_in_thread` / `self.after(0, …)` — see AlertsPage-style log marshalling (`on_log=lambda line: self.after(0, append, line)`). Never "fix" by adding locks around Tk calls.
2. **Selftest fails only on Windows CI with `'charmap' codec can't encode`**: something printed ₹/·/— to a cp1252 pipe. Keep `_utf8_console()` first in `selftest.run`; don't print non-ASCII from other CLI paths without it.
3. **Ruff import-sort failures only in CI**: first suspect a **missing file in the repo** (unresolvable first-party import degrades isort grouping) — verify the tree on GitHub before touching imports. `known-first-party = ["finsight"]` is already pinned; keep it.
4. **Pandas memory spikes** (reconciliation / DQ on big files): the DQ engine already streams via `profile_csv(chunksize=…)` with hard caps; if a new feature loads whole frames, copy that accumulator pattern (update per chunk → finalize) instead of raising machine specs. Grids only render `page_size` rows by design — exports use the full frame; do not "fix" the banner by rendering everything.
5. **PyInstaller build issues**: `ModuleNotFoundError` in frozen exe → add to `hiddenimports` in `FinSight.spec`. Plugins must be discovered via importable package names (pkgutil), never filesystem globs — already handled; don't regress. CustomTkinter theme JSONs come from `collect_data_files("customtkinter")`.
6. **Matplotlib in Tk**: build `Figure` objects (never pyplot state machine), draw on the Tk thread only (`draw_idle()` after computing data off-thread), `tight_layout()` before draw. Pattern: `analytics_page.py`, `mis_builder_tab.py`.
7. **CTkTabview quirk**: its `command` fires on tab change with no args; page-level `on_show` must be forwarded manually (see `mis_studio_page._tab_changed`). CTkOptionMenu values must be re-`configure(values=…)`d AND `.set()` after dynamic reloads.
8. **Scheduler bugs**: reproduce with `compute_next_run`/`run_due_jobs(now=datetime(...))` — everything is injectable; never debug through the daemon thread. Feb-31 style bugs are covered by `test_monthly_clamps_short_months`.
9. **Version bumps touch 4 files**: `pyproject.toml`, `src/finsight/__init__.py`, `installer/finsight.iss`, `scripts/build_portable.bat`. CI does not check this — grep before tagging.
10. **v1.5.0 tag is a broken snapshot** (missing the five MIS Studio engine files — failed web upload). Never diff/build against it; v1.5.1 is the canonical MIS Studio release.

## 6. Mock Data Simulation & Execution Guide

1. `python mock_data_simulator.py` — run from the repo root (needs pandas/numpy/openpyxl only). Writes `mock_data/lending_book.xlsx`, `lending_book.csv` and the deliberately hostile `lending_book_stress.csv`. Seeded (`SEED = 20260726`): identical output every run, so planted edge cases sit on stable rows.
2. `finsight` (installed) or `python -m finsight` from an installed source tree — open **MIS Studio** in the sidebar.
3. **One-Click Templates** tab → pick `mock_data/lending_book.xlsx` → each of the three giant buttons must produce a styled Excel. **Report Wizard** tab → same file → any pivot in three clicks. Then upload `lending_book_stress.csv`: messy headers, a blank row, duplicate rows and text-formatted numbers must yield a report, never a traceback.
4. Headless engine smoke (how these files were validated on 2026-07-26): from the repo root run `run_template` for all three template keys plus `build_pivot(group_by="branch", value="loan_amount")` against the xlsx — all green, TOTAL row present.

Planted edge cases: NaN outstanding/dpd, zero-amount and Rs 5,000,000 outlier loans, a negative adjustment, dpd 999, a blank and a Bengali customer name, and every DPD `pd.cut` bucket edge (0/1/29/30/31/59/60/61/89/90/91/180/365).

## 7. Release history quick map
v1.0 core suite → v1.1 MIS catalog/branding → v1.2 recon root-cause → v1.3 Windows packaging+docs → v1.4 Data Quality + API Explorer + plugin architecture → v1.5/**v1.5.1** MIS Studio (Builder/Templates/Auto-Reporter). CHANGELOG.md is accurate and maintained — keep it that way.
