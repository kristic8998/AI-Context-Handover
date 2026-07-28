# LendOps Toolkit - System Architecture & AI Handover Context

Repo: https://github.com/kristic8998/lendops-toolkit · Current: **v1.1.0** · Package `lendtoolkit`, CLI `lendtoolkit` (`--selftest` for headless verification). CI green on Ubuntu + Windows, Python 3.10/3.12.

## 1. Core Purpose & Business Value

Three heavy-duty lending utilities in one lightweight, 100% click-driven Windows desktop app for a micro-lending ops team: **BureauFlow Extractor** (parse CIBIL/Experian credit bureau reports — XML or text-PDF — into score/accounts/overdue/defaults, export Excel/CSV), **LedgerSync Pro** (3-way reconciliation: payment gateway vs bank statement vs internal DB, every transaction classified, pie summary, mismatch workbook), **AlertForge Engine** (DPD-triggered personalised collection reminders with escalation tiers, smtplib sender in **simulation mode by default**, live log, exportable trigger list). Every page has a permanent 3-step How-to-Use card and a "Try with sample data" button.

## 2. Tech Stack & Dependencies

- customtkinter, pandas, numpy, openpyxl, **pypdf** (lazy-imported inside the PDF path only), **tkinterdnd2** (drag-and-drop; graceful fallback to click-to-browse if missing). No matplotlib — the recon pie is drawn on a plain `tk.Canvas` (`PieChart` widget) to stay light.
- 45 pytest tests; ruff/black; headless selftest (4 checks) with the UTF-8 console shim; standard CI matrix; `requirements.txt` mirrors pyproject (owner requested both).
- Packaging: `LendOpsToolkit.spec` — note it collects **both** `collect_data_files("customtkinter")` and `collect_data_files("tkinterdnd2")`; the tkdnd binaries line is what makes drag-and-drop work in the frozen exe. `scripts/build_windows.bat`, `build_portable.bat`, `installer/lendops_toolkit.iss` (per-user, fixed AppId).

## 3. Complete Directory Structure

```
lendops-toolkit/
├── src/lendtoolkit/
│   ├── __init__.py  __main__.py  app.py  selftest.py
│   ├── core/        # paths.py (%LOCALAPPDATA%/LendOpsToolkit, LENDTOOLKIT_HOME),
│   │                # config.py (JSON: theme/start_page/dpd_threshold), tasks.py (TaskRunner),
│   │                # demo.py (seeded sample data incl. injected fraud/breaks)
│   ├── modules/     # PURE engines: tabular.py (read_table/find_col/num_series/text_series),
│   │                # bureau.py, ledgersync.py, alerts.py
│   └── ui/          # widgets.py (HelperCard, DropZone, PieChart, KpiCard, DataGrid w/ severity
│                    #   tinting, Toast, run_in_thread), pages/ (home, bureau, ledgersync, alerts)
├── tests/           # test_bureau / test_ledgersync / test_alerts / test_core
├── sample_data/     # bureau_report.xml, recon_gateway/bank/internal.csv, daily_loans.csv
├── scripts/  installer/  LendOpsToolkit.spec  launcher.py  requirements.txt  docs/
```

## 4. Architectural Blueprint

### 4.1 App shell & drag-and-drop
`app._make_base()` returns a DnD-capable root when tkinterdnd2 imports (`class _DnDRoot(ctk.CTk, TkinterDnD.DnDWrapper)` + `TkinterDnD._require(self)`), else plain `ctk.CTk`; `app.dnd_available` flags capability. `DropZone` registers `DND_FILES` only when available and always keeps click-to-browse; dropped paths parsed via `self.tk.splitlist(event.data)` (handles brace-wrapped paths with spaces). Nav table `_PAGES`, lazy `PAGE_FACTORIES`, Ctrl+D theme, Ctrl+1..4 pages, `_on_close` → runner shutdown + config save. Logging goes to `%LOCALAPPDATA%/LendOpsToolkit/logs/lendtoolkit.log`.

### 4.2 BureauFlow (`modules/bureau.py`)
Two parsing paths, one `_finalize()` → `BureauReport(applicant, score, score_band, accounts_df, active_loans, total_overdue, past_defaults, notes)`.
- **XML** (`parse_xml_text`): schema-agnostic tag-hint parser. Score = first leaf tag containing "score" with a 300–900 value. An *account node* = tag containing account/tradeline/loan **with children** and **no account-like child that itself has children** (`_is_account_node` — this container-vs-record test fixed a real duplicate-row bug; keep it). Fields mapped by `_FIELD_HINTS` (lender/type/sanctioned/balance/overdue/dpd/status).
- **PDF** (`_parse_pdf` → `extract_from_text`): pypdf text extraction + regex layer — score patterns, `Name:` capture, and `Lender:`-delimited field blocks. Empty text raises a friendly "scanned/image PDF" `ValueError`. **No OCR by design** — report honestly instead.
- Derived: active = status contains active/open/current/live; past_defaults = status contains written-off/settled/suit/wilful/default OR dpd ≥ 90. Bands: 750+/700+/650+ → Excellent/Good/Fair else Poor.

### 4.3 LedgerSync Pro (`modules/ledgersync.py`) — fully vectorized
`_normalize(frame, source)` extracts (txn_id, amount) per source via hints (`txn_id/utr/reference/id`, `amount/credit/debit`) with friendly per-file errors. In-source duplicate ids are counted, flagged (`Duplicate ID`) and deduped keep-first for matching. Outer-merge on txn_id → presence masks → one `np.select` ladder assigns `STATUS_ORDER`: Amount Mismatch / Pending Settlement (gateway+internal, no bank) / Missing in Internal DB / Missing in Gateway / Only in X / Duplicate ID / Success (all three within `amount_tolerance`, default ₹1). No per-row Python loops anywhere — keep it that way for high-volume files. `ReconResult.mismatch_frame` drives the UI grid; export = Mismatches / All Transactions / Summary sheets.

### 4.4 AlertForge (`modules/alerts.py`)
`build_alerts(df, AlertRule(dpd_threshold, gentle_max=15, firm_max=45))` → filter dpd ≥ threshold, tier Gentle/Firm/Final, personalised message from `_TEMPLATES` (name/amount/loan_id/dpd interpolated), sorted worst-first. `EmailSender(dry_run=True)` **cannot send by accident** — real SMTP requires explicit host+sender and unchecking Simulation mode in the UI; `send_batch` never raises per-row (skipped/failed rows are logged via `on_log` callback, which the UI marshals with `self.after(0,…)`). Export: Alerts + Summary sheets (or CSV).

 **Since v1.1.0:** every engine coerces numeric columns through the app's single robust parser (`parse_amount_series` — text money "Rs 1,20,000.00", accounting negatives, Cr/Dr suffixes, percents) instead of bare `to_numeric`; never regress to plain `to_numeric` on user-supplied columns. LedgerSync `_normalize` also uses it per leg.

## 5. AI-to-AI Debugging Heuristics

1. **Drag-and-drop dead, clicking works**: tkdnd binaries missing. From source → `pip install tkinterdnd2`; in a frozen exe → the spec lost `collect_data_files("tkinterdnd2")`. The app must keep degrading gracefully — never make DnD load-bearing.
2. **CTk threading errors**: same contract as all sibling apps — worker→UI only via `run_in_thread`/`after(0)`. The AlertForge live log is the reference implementation for streaming logs from a worker.
3. **Bureau XML yields duplicated/ghost accounts**: a new bureau format has nested account containers — extend `_is_account_node`, don't loosen it. Score not found: verify a 300–900 leaf under a "score"-ish tag exists; never widen to 4-digit matches (years match).
4. **PDF returns empty/garbage**: check `extract_text()` output first — scanned PDFs are unsupported by design (friendly error exists). For new text layouts, extend `_BLOCK_FIELD_RES` regexes and add a text fixture test to `TestTextExtraction`.
5. **Recon slow or memory-heavy on huge files**: someone added a row-loop or `.apply` — restore vectorized masks/`np.select`. Legit-fee mismatches: raise `amount_tolerance` per run, don't hardcode.
6. **"Could not find a transaction-id column in the bank file"**: working as intended — the error names the file; extend `_ID_HINTS`/`_AMOUNT_HINTS` only for genuinely common header names.
7. **Emails sent in tests?** Impossible unless someone constructed `EmailSender(dry_run=False, host=…)` — tests must only use dry_run or the no-host skip path (`test_real_mode_without_host_skips_safely`).
8. **Windows selftest encoding** → `_utf8_console()` shim, same as FinSight.
9. Version bump touches: pyproject, `__init__.py`, `installer/lendops_toolkit.iss`, `scripts/build_portable.bat`.

## 6. Mock Data Simulation & Execution Guide

1. `python mock_data_simulator.py` — repo root; writes `mock_data/bureau_report.xml`, `recon_gateway.csv` / `recon_bank.csv` / `recon_internal.csv` (headers deliberately differ per leg), `daily_loans.xlsx`. The script **prints every planted break with its transaction id**.
2. `lendtoolkit` (or `python -m lendtoolkit`) → load each file on its page.
3. Expected results: **BureauFlow** — score 716 (Good), exactly **4** tradelines (the `<Accounts>` container must not duplicate), total overdue **19,200**, 1 past default; amounts are comma-formatted on purpose. **LedgerSync** — 200 txns, 195 Success (97.5%), and `UTR300001→Amount Mismatch, 02→Pending Settlement, 03→Missing in Internal DB, 04→Missing in Gateway, 05→Duplicate ID, 06→Success (0.50 within tolerance)`. **AlertForge** — dpd values sit on the 15/16 and 45/46 tier boundaries; dry-run mode means nothing sends.
4. Headless validation (2026-07-26): all expectations above verified green against the real engines before publishing.

**v1.0.1 exists because of this simulator**: its comma-formatted bureau amounts exposed a real bug — `_finalize` used plain `to_numeric`, silently zeroing "50,000"-style values (the repo's own comma-less sample never triggered it). Fixed via `_to_number` mapping + regression test `test_comma_formatted_amounts_parse`. Moral: mock data must be *nastier* than the sample data.
