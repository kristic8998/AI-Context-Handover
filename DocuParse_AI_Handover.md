# DocuParse AI - System Architecture & AI Handover Context

**Not a standalone repo.** DocuParse AI is a module inside **FinOps Command Center** — https://github.com/kristic8998/finops-command-center (**private**), v1.0.0, built in a parallel Cowork session. This document is derived from that session's own handoff records (`SESSION_HANDOFF.md` + `FILE_MAP.md`, provided by the owner on 2026-07-26), not from a first-hand code read by this architect. Where this doc and the actual code disagree, the code wins — verify before large refactors. **Corrections exist:** a first-hand document by the architect who built this app was later added — [FinOps_Command_Center_Handover.md](FinOps_Command_Center_Handover.md) (2026-07-27). Where the two disagree, prefer it; prefer the code over both.

## 1. Core Purpose & Business Value

Turns PDF bank statements and scanned/photographed documents into structured, styled Excel: extracted tables plus regex-mined fields (PAN, IFSC, GSTIN, account numbers, balances). Handles the real-world worst case — a photographed statement — by rebuilding tables from OCR word boxes. **PII masking is ON by default** (`XXXXXXXX7890`, `A**** S*****`, last four characters survive for reconciliation); there is a UI toggle to disable. Rationale recorded in the source session: the team's Claude plan has no BAA, so exports must not casually carry full identifiers. All test data was synthetic.

## 2. Tech Stack & Dependencies

pdfplumber, pypdf, pytesseract, pdf2image, opencv-python-headless (optional), pandas, numpy, openpyxl, XlsxWriter. **External programs Tesseract OCR and Poppler are NOT bundled** into the exe — installed per machine; the app's Dashboard probes what's missing and names the affected feature. Host app: customtkinter shell, PyInstaller `build.spec` (folder mode), hardened `run.bat` launcher. **Now tested + CI green** (per the first-hand doc, 2026-07-27): 156 pytest tests, CI with lint / 4-way test matrix (Ubuntu+Windows × Python 3.11/3.12) / packaging-sanity / advisory forward-compat jobs. Still true: **the GUI has never been rendered on a real screen** (mock-Tk verification only — see §5.1).

## 3. Complete Directory Structure (module slice of finops-command-center)

```
app/modules/docuparse/        # PURE engine — no Tkinter anywhere (enforced by AST check)
├── models.py      (285 ln)   # ParseOptions, ExtractedTable, DocumentResult, BatchResult
├── ocr.py         (314)      # Tesseract facade + preprocessing chain
├── extractor.py   (484)      # 3-tier table extraction w/ OCR word-box fallback
├── cleaner.py     (371)      # header promotion, coercion, parse_number (shared money parser)
├── field_miner.py (338)      # regex field mining + PII masking
├── exporter.py    (316)      # styled multi-sheet Excel
└── pipeline.py    (123)      # extract → clean → mine orchestration
app/ui/pages/docuparse_page.py (842)   # UI, incl. drag-and-drop via app/ui/dnd.py
```

## 4. Architectural Blueprint

- **Pipeline** (`pipeline.py`): extract → clean → mine; result objects from `models.py` flow to `exporter.py` → Excel with Summary / Fields / Combined / one-sheet-per-table.
- **Extraction ladder** (`extractor.py`): pdfplumber ruled-lines → retry with text-alignment strategy → OCR fallback that **rebuilds tables from OCR word boxes using vertical gap projection** (this is what makes photographed statements work).
- **OCR preprocessing** (`ocr.py`): greyscale → denoise → adaptive threshold → deskew, deskew only corrects tilts <15° (larger corrections do more harm than good).
- **Cleaner** (`cleaner.py`): header-row promotion; `_drop_repeated_headers()` drops rows where ≥60% of populated cells equal the column name (multi-page PDF header repeats); numeric coercion accepts n≥1 populated values when 100% of them parse (sparse Debit/Credit columns); **`parse_number()` is the single shared money parser for the whole app** — handles ₹/$/€ symbols AND currency words ("Rs 15,000.00") via a digit-lookahead regex, plus accounting negatives `(2,500.00)` → `-2500.0`. House rule: never write a second money parser; the MIS modules import this one.
- **Field miner** (`field_miner.py`): keyword-anchored regexes — national-ID matching requires `aadhaar|uid|national id` context (a bare 12-digit run also matches account numbers); masking applied before anything reaches a spreadsheet.
- **Host-app contracts**: engines never import Tkinter; work runs through `BackgroundTaskRunner` and returns via `UiDispatcher.post` (note: names differ from the FinSight/LendOps `TaskRunner`/`run_in_thread` but the contract is identical); every page carries the mandatory 3-step How-to-Use card.

## 5. AI-to-AI Debugging Heuristics

1. **The GUI has never been rendered on a real screen** — the single biggest open risk of the whole host app. All UI verification used a mock customtkinter/tkinter stub. First real launch may surface CTk version arg differences or layout surprises; crashes land in `logs/finops.log` + a dialog. When the owner pastes a traceback: analyze → corrected code → explain in Bengali. Do not apologize.
2. **Amounts stay text / never totalled**: currency *words* are the usual culprit — check `parse_number()`'s `_CURRENCY_WORD` regex before anything else ("Rs"/"INR" prefixes; verified against 20 cases; "Resolved"/"USDA grant" must stay rejected).
3. **Ghost duplicate header row in output** (and a text-typed numeric column as its side-effect): repeated multi-page header — the ≥60%-cells-match rule in `_drop_repeated_headers()` is the fix point; `drop_duplicates` cannot catch it.
4. **National-ID false positives on account numbers**: keep the keyword anchor; never loosen back to a bare 12-digit pattern.
5. **OCR path returns nothing**: Tesseract/Poppler missing on the machine — check the Dashboard probes first, not the code.
6. **New deferred import added anywhere** → add it to `build.spec` `hiddenimports` or the frozen exe dies with `ModuleNotFoundError`.
7. Money parsing, scheduling, page registration: reuse the existing single implementations (`parse_number`, `ScheduleSpec`+`CronusScheduler`, `PageSpec` in `app/ui/registry.py`) — the host app's core discipline is "one engine, many views".

## 6. Mock Data Simulation & Execution Guide

`mock_data_simulator.py` sits at the **finops-command-center repo root** (added 2026-07-26 by the outgoing architect; generation logic verified standalone — the *app-side* run still needs a first real pass, since this architect never had the app's code locally).

1. `python mock_data_simulator.py` — repo root; needs pandas/numpy (+ `reportlab` for the PDF; skipped with a printed note if absent). Seeded, reproducible.
2. `python main.py` (source) or the frozen exe → use the files below on the matching pages.

3. **DocuParse AI page** → `mock_data/statement.pdf`. Expected: 7 transaction rows; the repeated mid-table header row must be dropped by `_drop_repeated_headers()`; `Debit/Credit/Balance` coerce to float64 (sparse columns allowed); `"Rs 42,000.00"` parses via `parse_number()`'s currency-word branch; `"(2,500.00)"` → `-2500.0`; the export must show PII **masked** (`XXXXX...4567`, PAN masked) because masking is default-on.
4. **Verify, don't eyeball**: `mock_data/statement_expected.csv` is the ground truth for the PDF's transaction table — the extracted table must match it row for row after cleaning. (Note: MIS Studio is NOT part of this repo — it was removed; `finsight` v1.5.1 owns it. Earlier drafts of this doc referenced MIS loan-book files; they now live conceptually with finsight's own simulator.)
5. Related engine trap worth knowing before touching `cleaner.py`: the pandas-3 `StringDtype` dtype-guard bug — see the first-hand doc's §5 heuristics.
