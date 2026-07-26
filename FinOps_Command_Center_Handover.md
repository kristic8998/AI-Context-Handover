# FinOps Command Center — AI Handover Document

**First-hand.** Unlike `DocuParse_AI_Handover.md` and `Cronus_Orchestrator_Handover.md`,
which were reconstructed from session records, this document was written by the architect who
built the application. Where it disagrees with those two, prefer this one — and prefer the code
over all three.

**Repo:** `github.com/kristic8998/finops-command-center` (private) · v1.0.0 · CI green (run #21)

**Corrections to the existing README and module docs, as of 2026-07-27:**

- The table row "no CI yet" is stale. There is now CI: lint, a 4-way test matrix
  (Ubuntu + Windows × Python 3.11 + 3.12), a packaging-sanity job and an advisory
  forward-compatibility job. All green.
- "no committed test suite" is stale. There are **156 pytest tests**, all synthetic fixtures.
- The cross-project quality gate in the README says `src`; this project uses a **flat `app/`
  package**, and its Python floor is **3.11**, not 3.10.
- MIS Studio is **not** part of this repo. It was built here once by mistake and has been
  removed; `finsight` v1.5.1 owns it. See §5.
- Still true and still important: **the GUI has never been rendered on a real screen.** Every
  UI verification went through a mock Tk layer, because the build sandbox has no `tkinter`.
  Treat all layout, focus order and DPI behaviour as unverified.

---

## 1. Core Purpose & Business Value

One window replacing two manual routines for a digital micro-lending team.

**DocuParse AI** turns PDF bank statements and scanned/photographed KYC documents into a
styled multi-sheet Excel workbook: extracted tables plus regex-mined fields (PAN, IFSC, GSTIN,
account number, statement period, opening/closing balance). Sensitive identifiers are **masked
by default** — an operations analyst should not be able to produce an unmasked export by
accident.

**Cronus Orchestrator** registers `.py` / `.pyw` / `.bat` / `.cmd` / `.ps1` scripts and runs,
schedules and monitors them with live streaming output, so recurring reconciliation jobs stop
living in someone's head.

The user is a non-technical operations person on a locked-down Windows 11 laptop, 16 GB RAM,
no GPU. Two consequences that drive the whole design: **every screen carries a permanent
three-step "How to Use" card**, and **the window must never freeze** — all real work is on
background threads.

## 2. Tech Stack, Libraries & CI/CD

CustomTkinter · pandas · pdfplumber + pypdf · pytesseract + pdf2image + OpenCV (headless) ·
openpyxl + XlsxWriter · APScheduler + tzlocal · psutil · Pillow · PyInstaller.

Tesseract and Poppler are **external binaries, deliberately not bundled**. The app starts and
stays usable without them; Dashboard → Environment health names exactly what is missing and
which feature it disables.

**Quality gate** (all green locally and in CI):

```bash
ruff check app tests main.py
black --check app tests main.py
pytest tests/ -q                                    # 156 tests
FINOPS_TEST_FUTURE_STRINGS=1 pytest tests/ -q       # same suite, pandas 3 string dtype
python main.py --selftest                           # 13 headless checks
python -m compileall -q app main.py
```

`--selftest` is the load-bearing one: it is what catches a hidden import missing from
`build.spec`, which would otherwise ship an `.exe` whose window silently never appears.
`scripts/build_windows.bat` runs it against the **packaged** executable too and refuses to
build on failure.

CI jobs: **lint** · **test** (matrix; installs from `requirements.txt`, i.e. what users
actually get) · **packaging-sanity** (AST-scans every deferred import and fails if it is absent
from `build.spec` hidden imports) · **forward-compat** (newest release of every dependency,
`continue-on-error: true`).

ruff deliberately selects `BLE` and `PLC0415`. This codebase makes exactly two architectural
exceptions — broad excepts at UI boundaries, and imports inside functions — and they must be
justified with a `# noqa` at each site rather than hidden behind a project-wide ignore.

## 3. Directory Structure

Flat `app/` package, **not** `src/`.

```
main.py                  Entry point; --selftest / --version / --help before any GUI work
app/application.py       Root window, navigation, page lifecycle, shutdown
app/core/                config · logger · settings · concurrency   (framework-agnostic)
app/modules/docuparse/   models · ocr · extractor · cleaner · field_miner · exporter · pipeline
app/modules/cronus/      models · registry · runner · scheduler
app/selftest.py          13 headless checks; exit code = failure count
app/ui/                  theme · registry · sidebar · dnd · widgets/ · dialogs/ · pages/
tests/                   156 tests, synthetic fixtures only
build.spec  scripts/  installer/  docs/  .github/workflows/ci.yml
```

## 4. Architectural Blueprint

**Invariant 1 — `app/modules/` never imports Tkinter.** Each engine is a plain library,
runnable from a console and reusable in other scripts. This is machine-checked, both by a test
and by a selftest check; it is not a convention you may quietly break.

**Invariant 2 — only the Tk thread touches widgets.** `BackgroundTaskRunner` is a bounded
`ThreadPoolExecutor`; results return through `UiDispatcher`, a thread-safe queue drained by Tk
`after()`. `TaskContext` carries cooperative cancellation and progress. A worker thread that
touches a widget will appear to work and then crash days later.

**Invariant 3 — pages and heavy dependencies are imported lazily.** The page registry holds
`"module.path:ClassName"` strings and uses `probe_importable()`, so a missing optional package
greys out one nav entry instead of preventing startup. This is *why* `build.spec` needs an
explicit `hiddenimports` list: PyInstaller's static analyser only follows module-level imports,
and almost nothing here is imported at module level.

**Invariant 4 — nothing crashes the shell.** Page construction, background tasks, log sinks and
callbacks are each individually guarded. `BasePage.build_failed` exists so the shell skips
lifecycle hooks for a page that failed to build — without it, `on_show()` ran against a
half-constructed page.

Persistence is atomic JSON (temp file + `os.replace`) with **corrupt-file quarantine**: a
truncated settings file is moved aside and defaults are loaded, rather than taking the app down.

Extraction path: pdfplumber ruled-lines → retry with text-alignment when ruled lines find
nothing → OCR fallback for scans, reconstructing tables from Tesseract word boxes by **vertical
gap projection**. Process control uses psutil **process-tree** termination (escalating terminate
→ kill), `python -u` plus `PYTHONUNBUFFERED=1` for genuinely live streaming, and
`CREATE_NO_WINDOW` so scheduled runs do not flash a console.

## 5. AI-to-AI Debugging Heuristics

**Never test a pandas dtype by comparing against `object`.** The two coercion guards in
`cleaner.py` read `if series.dtype != object: continue`. pandas 2 stores text as `object`, but
pandas 3 stores it as `StringDtype` — so on pandas 3 both guards skipped **every** column:
amounts stayed text, dates stayed text, nothing was ever totalled, and the export looked
perfectly plausible. This is the most expensive bug class in this codebase: silent, plausible,
wrong. Always test for the type you are converting **to**
(`is_numeric_dtype` / `is_datetime64_any_dtype`).

**Corollary — a regression test can pass and still prove nothing.** The obvious test, calling
`astype("string")` on the input frame, passes against the *broken* code, because header
promotion rebuilds the frame and resets the dtype to `object`. Only toggling the global
`pandas.options.future.infer_string` reproduces it. Before trusting any regression test, revert
the fix and confirm the test actually fails.

**Pin CI to `requirements.txt`, not to unpinned installs — but keep one unpinned job.** The
test matrix originally installed dependencies unpinned, silently picked up pandas 3, and went
red; the failure read like a CI quirk rather than the real product bug it was. The fix is both
halves: the matrix now installs the ranges users get, and a separate advisory `forward-compat`
job keeps the early-warning signal. That job is what caught this, and it now passes against
**pandas 3.0.5**.

**Do not build the same feature twice.** MIS Studio was built here because no `finsight` code
was visible in the session, so "Finsight" was assumed to mean this app. Both implementations
worked; their DPD bucket edges differed (6 buckets here, 5 in `finsight`), so the same portfolio
would age differently depending on which tool ran. Removed from this repo. When a directive
names a product you cannot see, confirm which repo it means before writing code.

**Deleting files locally is not deleting them on GitHub.** MIS removal was done in the working
copy, but six files stayed on `main` — and produced three failing CI jobs whose messages pointed
at lint rules and hidden imports rather than at the real cause. After any web-UI restructuring,
**diff the remote tree against the local tree** before reading a single CI log.

**GitHub web-uploader specifics** (this repo is published entirely through the browser; there is
no git CLI and no token — never ask for one):

- Uploads are asynchronous. Clicking Commit while files are still uploading silently does
  nothing. Wait for the file list to settle.
- Commit buttons need a **coordinate** click; clicking by element reference does not submit.
- The **first** click after a fresh page load is absorbed during hydration — click, screenshot
  to confirm, then click again. Do not blind-retry: a second click on an *open* dialog closes
  it, so retries can toggle it shut and look like a failure.
- `find` / accessibility-tree results go stale on these pages. Trust screenshots.
- **Never type source code into the web editor.** Synthetic typing drops characters
  (`"tables"` → `"ables"`). Always upload the file.
- `git init` inside a Cowork mount produces a broken `.git` — the mount blocks `unlink`, which
  git needs for lock files, so `index.lock` goes stale.

**Windows-specific traps already handled; keep them.** `.gitattributes` marks `*.bat` as
`-text` (not `eol=crlf`) because `eol=crlf` only applies on checkout, and a ZIP download from
the web page never checks out — LF-only batch files make `cmd.exe` mis-seek on `goto`. Windows
CI pipes are cp1252, so selftest output goes through
`stream.reconfigure(encoding="utf-8", errors="replace")`. `.gitignore` uses `exports/*`, not
`exports/`, because git cannot re-include a `.gitkeep` beneath an excluded directory.

**Two engine details worth not re-breaking.** The money parser accepts currency written as a
*word* (`Rs 1,20,000.00`, `INR 2,500`), guarded by a digit lookahead so `Resolved` and
`USDA grant` are still rejected. The national-ID regex is keyword-anchored (`aadhaar|uid|
national id`) because an unanchored 12-digit pattern matched ordinary account numbers and masked
them as national IDs.

**All test data is synthetic, deliberately.** This project handles lending data under a Claude
Team plan with no BAA, so no real customer record may enter a prompt or a fixture. Keep it that
way when adding tests.
