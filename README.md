# AI-Context-Handover

Architectural handover documentation from the outgoing AI architect (Claude) to any incoming AI assistant (e.g. Gemini) working on Kristi Chakraborty's projects (github.com/kristic8998). Each document is written **for an AI reader**: it front-loads the mental model, the invariants you must not break, and battle-tested debugging heuristics learned while actually building and shipping these systems.

## Documents

| File | Project | Repo | Status |
|---|---|---|---|
| [Finsight_Handover.md](Finsight_Handover.md) | FinSight — enterprise lending analytics desktop suite (incl. MIS Studio: Visual Builder, One-Click Templates, Auto-Reporter) | [finsight](https://github.com/kristic8998/finsight) | v1.5.1, CI green |
| [AIR_Handover.md](AIR_Handover.md) | AIR — Automation Intelligence Recorder (workflow → generated Python + SOP docs) | [air](https://github.com/kristic8998/air) | v0.1.0, CI green |
| [LendOps_Toolkit_Handover.md](LendOps_Toolkit_Handover.md) | LendOps Toolkit — BureauFlow, LedgerSync Pro, AlertForge | [lendops-toolkit](https://github.com/kristic8998/lendops-toolkit) | v1.0.1, CI green |
| [LendOps_Studio_Handover.md](LendOps_Studio_Handover.md) | LendOps Studio — Collecta, PolicySim, KYC Sentinel | [lendops](https://github.com/kristic8998/lendops) | v1.0.0, CI green |
| [DocuParse_AI_Handover.md](DocuParse_AI_Handover.md) | DocuParse AI — PDF/scan → structured Excel (module of FinOps Command Center) | [finops-command-center](https://github.com/kristic8998/finops-command-center) (private) | v1.0.0, CI green |
| [Cronus_Orchestrator_Handover.md](Cronus_Orchestrator_Handover.md) | Cronus Orchestrator — script run/schedule/monitor (module of FinOps Command Center) | [finops-command-center](https://github.com/kristic8998/finops-command-center) (private) | v1.0.0, CI green |
| [FinOps_Command_Center_Handover.md](FinOps_Command_Center_Handover.md) | FinOps Command Center — host app for the two modules above (**first-hand**; prefer it where docs disagree) | [finops-command-center](https://github.com/kristic8998/finops-command-center) (private) | v1.0.0, CI green, 156 tests |

## Honest scope note (read this first)

**Update 2026-07-26:** DocuParse_AI and Cronus_Orchestrator were initially omitted because no standalone repos by those names exist. The owner has since provided the build session's own handoff records (`SESSION_HANDOFF.md` + `FILE_MAP.md`) showing both exist as **modules inside the private `finops-command-center` repo**, built in a parallel session. Their handover docs are now included, **derived from those records rather than a first-hand code read** — each doc states this provenance in its header; treat the code as authoritative where they disagree. FinOps Command Center now has 156 tests and green CI (see its first-hand handover doc, added 2026-07-27), but its GUI has still never been rendered on a real screen — verified only via a mock-Tk layer.

Other real repositories not (yet) documented here: `filesmith` (CLI file organizer), `visionqc` (vision QC library/API), `xlforge` / `xlforge-web`.

## Cross-project conventions (apply everywhere)

The four repos above share one architecture DNA. Learn it once. (FinOps Command Center follows the same *contracts* with different names: `BackgroundTaskRunner`/`UiDispatcher.post` instead of `TaskRunner`/`run_in_thread`, `ColumnMatcher` concept-aliases instead of `find_col` hints — details in its two module docs.)

1. **Engine/UI split.** Business logic lives in `modules/` as pure pandas/stdlib services with **zero UI imports** — every engine is unit-testable headless. UI pages are thin CustomTkinter frames that call engines through a thread pool.
2. **Threading contract.** All heavy work goes through `TaskRunner.submit(fn, on_done=…, on_error=…)` (a small `ThreadPoolExecutor` wrapper). UI callbacks are marshalled back to the Tk thread via `widget.after(0, …)` — the helper `run_in_thread(widget, runner, func, on_done, on_error)` in each project's `ui/widgets.py` does this. **Never touch a Tk widget from a worker thread.**
3. **Forgiving column detection.** Engines resolve DataFrame columns by name *hints* (`find_col(df, ("dpd", "past_due", …))` — exact match wins, then substring). This is a deliberate product decision: laymen upload arbitrary LMS exports. Do not replace it with strict schemas.
4. **Friendly failure.** Engines raise `ValueError`/domain errors with plain-English, user-facing messages; UI shows them via toast or `FriendlyDialog` popups. A crash reaching the user is a bug by definition.
5. **Quality gate before any publish:** `ruff check src tests` + `black --check` + `pytest` + headless `--selftest` + `python -m compileall src`, all green locally **and** in GitHub Actions (Ubuntu + Windows, Python 3.10/3.12).
6. **Windows packaging standard** (per the owner's standing rule): every desktop app ships a PyInstaller one-folder spec, `scripts/build_windows.bat` + `build_portable.bat`, an Inno Setup script under `installer/`, and INSTALL/USER_GUIDE/TROUBLESHOOTING docs. PyInstaller cannot cross-compile — exe builds happen on the owner's Windows 11 laptop (16 GB RAM, integrated graphics: keep everything CPU-light).
7. **Mandatory mock-data simulator** (owner's standing rule, 2026-07-26): every project ships a standalone `mock_data_simulator.py` at the repo root — pandas/numpy, seeded/reproducible, generating realistic **edge-case-filled** datasets tailored to that app's engines, with planted cases on known rows/ids so results can be *verified*, not eyeballed. Every README gets a non-technical "Test-drive with realistic mock data" section (simulator first, then the app). Every handover doc in this repo carries a matching **§6 Mock Data Simulation & Execution Guide**. Make the mock data nastier than the sample data — the Toolkit's simulator caught a real comma-parsing bug (v1.0.1) on its first run.

## Hard-won operational heuristics (meta)

- **GitHub web-uploader can fail silently.** During the v1.5.0 release, one folder commit landed *empty* (five engine files missing) while later commits succeeded; CI first surfaced it as a *misleading lint error* (ruff couldn't resolve the missing first-party module, so import-sorting "broke"). Lesson: after web-uploading, verify the tree (open the folder listing), and treat "weird lint failures only in CI" as a possible missing-file symptom. v1.5.1 is the repaired tag; never build from the v1.5.0 tag.
- **Windows CI pipes are cp1252.** Anything printed by selftests (₹, ·, —) must go through `stream.reconfigure(encoding="utf-8", errors="replace")` — already implemented in every project's selftest; keep it when adding checks.
- **Pin ruff's `known-first-party`** in `pyproject.toml` ([tool.ruff.lint.isort]) — resolution by filesystem heuristics differs between environments.
- **Commit the CI workflow file last** when bulk-uploading a new repo via the web UI: you get exactly one authoritative CI run on the complete tree instead of transient red runs.
