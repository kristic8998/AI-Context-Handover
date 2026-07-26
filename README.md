# AI-Context-Handover

Architectural handover documentation from the outgoing AI architect (Claude) to any incoming AI assistant (e.g. Gemini) working on Kristi Chakraborty's projects (github.com/kristic8998). Each document is written **for an AI reader**: it front-loads the mental model, the invariants you must not break, and battle-tested debugging heuristics learned while actually building and shipping these systems.

## Documents

| File | Project | Repo | Status |
|---|---|---|---|
| [Finsight_Handover.md](Finsight_Handover.md) | FinSight — enterprise lending analytics desktop suite (incl. MIS Studio: Visual Builder, One-Click Templates, Auto-Reporter) | [finsight](https://github.com/kristic8998/finsight) | v1.5.1, CI green |
| [AIR_Handover.md](AIR_Handover.md) | AIR — Automation Intelligence Recorder (workflow → generated Python + SOP docs) | [air](https://github.com/kristic8998/air) | v0.1.0, CI green |
| [LendOps_Toolkit_Handover.md](LendOps_Toolkit_Handover.md) | LendOps Toolkit — BureauFlow, LedgerSync Pro, AlertForge | [lendops-toolkit](https://github.com/kristic8998/lendops-toolkit) | v1.0.0, CI green |
| [LendOps_Studio_Handover.md](LendOps_Studio_Handover.md) | LendOps Studio — Collecta, PolicySim, KYC Sentinel | [lendops](https://github.com/kristic8998/lendops) | v1.0.0, CI green |

## Honest scope note (read this first)

Two additionally requested documents — **DocuParse_AI** and **Cronus_Orchestrator** — are **not included** because no such projects exist in this account's repositories or in the outgoing architect's build history. Writing an "architectural blueprint" for software that does not exist would poison an AI-to-AI handover with fabricated context. If these projects live elsewhere, share their source and a matching handover document can be produced to the same standard.

Other real repositories not (yet) documented here: `filesmith` (CLI file organizer), `visionqc` (vision QC library/API), `xlforge` / `xlforge-web`.

## Cross-project conventions (apply everywhere)

These four desktop apps share one architecture DNA. Learn it once:

1. **Engine/UI split.** Business logic lives in `modules/` as pure pandas/stdlib services with **zero UI imports** — every engine is unit-testable headless. UI pages are thin CustomTkinter frames that call engines through a thread pool.
2. **Threading contract.** All heavy work goes through `TaskRunner.submit(fn, on_done=…, on_error=…)` (a small `ThreadPoolExecutor` wrapper). UI callbacks are marshalled back to the Tk thread via `widget.after(0, …)` — the helper `run_in_thread(widget, runner, func, on_done, on_error)` in each project's `ui/widgets.py` does this. **Never touch a Tk widget from a worker thread.**
3. **Forgiving column detection.** Engines resolve DataFrame columns by name *hints* (`find_col(df, ("dpd", "past_due", …))` — exact match wins, then substring). This is a deliberate product decision: laymen upload arbitrary LMS exports. Do not replace it with strict schemas.
4. **Friendly failure.** Engines raise `ValueError`/domain errors with plain-English, user-facing messages; UI shows them via toast or `FriendlyDialog` popups. A crash reaching the user is a bug by definition.
5. **Quality gate before any publish:** `ruff check src tests` + `black --check` + `pytest` + headless `--selftest` + `python -m compileall src`, all green locally **and** in GitHub Actions (Ubuntu + Windows, Python 3.10/3.12).
6. **Windows packaging standard** (per the owner's standing rule): every desktop app ships a PyInstaller one-folder spec, `scripts/build_windows.bat` + `build_portable.bat`, an Inno Setup script under `installer/`, and INSTALL/USER_GUIDE/TROUBLESHOOTING docs. PyInstaller cannot cross-compile — exe builds happen on the owner's Windows 11 laptop (16 GB RAM, integrated graphics: keep everything CPU-light).

## Hard-won operational heuristics (meta)

- **GitHub web-uploader can fail silently.** During the v1.5.0 release, one folder commit landed *empty* (five engine files missing) while later commits succeeded; CI first surfaced it as a *misleading lint error* (ruff couldn't resolve the missing first-party module, so import-sorting "broke"). Lesson: after web-uploading, verify the tree (open the folder listing), and treat "weird lint failures only in CI" as a possible missing-file symptom. v1.5.1 is the repaired tag; never build from the v1.5.0 tag.
- **Windows CI pipes are cp1252.** Anything printed by selftests (₹, ·, —) must go through `stream.reconfigure(encoding="utf-8", errors="replace")` — already implemented in every project's selftest; keep it when adding checks.
- **Pin ruff's `known-first-party`** in `pyproject.toml` ([tool.ruff.lint.isort]) — resolution by filesystem heuristics differs between environments.
- **Commit the CI workflow file last** when bulk-uploading a new repo via the web UI: you get exactly one authoritative CI run on the complete tree instead of transient red runs.
