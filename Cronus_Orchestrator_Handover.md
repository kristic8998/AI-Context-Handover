# Cronus Orchestrator — AI Handover Document

**Not a standalone repo.** Cronus Orchestrator is a module inside **FinOps Command Center** — https://github.com/kristic8998/finops-command-center (**private**), v1.0.0, built in a parallel Cowork session. This document is derived from that session's own handoff records (`SESSION_HANDOFF.md` + `FILE_MAP.md`, provided by the owner on 2026-07-26), not from a first-hand code read by this architect. Where this doc and the actual code disagree, the code wins.

## 1. Core Purpose & Business Value

Register, run, schedule and monitor external scripts (`.py` / `.bat` / `.ps1`) from a click-driven desktop page with a live, terminal-style log viewer — so a non-technical ops team can operate the automation scripts an analyst writes, without a terminal. Its `ScheduleSpec` + `CronusScheduler` are also the scheduling backbone for the app's MIS Auto-Reporter (deliberate reuse — see §4.4).

## 2. Tech Stack, Libraries & CI/CD

APScheduler (+ tzlocal), psutil, stdlib `subprocess`. Host app: customtkinter, PyInstaller. `.bat`/`.ps1` execution is Windows-only by design; `.py` runs anywhere. **No committed test suite / CI yet** — the source session verified the runner against 10 live scenarios (success, non-zero exit, traceback capture, 2s timeout, mid-run stop with 11 streamed ticks, double-start refusal, missing file, unsupported type, cwd/env injection, unicode ₹/Bengali intact) and armed a real schedule that fired after 2.0s.

## 3. Directory Structure (module slice of finops-command-center)

```
app/modules/cronus/           # PURE engine — no Tkinter anywhere
├── models.py    (384 ln)     # ScriptEntry, ScheduleSpec (REUSED by MIS), RunRecord
├── registry.py  (239)        # persistent script registry (atomic JSON + corrupt-file quarantine)
├── runner.py    (463)        # async subprocess + live streaming + process-tree kill
└── scheduler.py (283)        # CronusScheduler — APScheduler wrapper
app/ui/pages/cronus_page.py      (643)  # orchestration UI + log viewer wiring
app/ui/dialogs/script_dialog.py  (478)  # add/edit script
app/ui/dialogs/schedule_dialog.py(457)  # "set an alarm" scheduling dialog
app/ui/widgets/log_viewer.py     (222)  # colour-tagged streams, block trimming, follow-tail
```

## 4. Architectural Blueprint

### 4.1 Runner (`runner.py`) — live output is the whole product
One reader thread **per stream** (stdout/stderr), never `communicate()`. Python children launched with `python -u` + `PYTHONUNBUFFERED=1` — without both, a child buffers ~8 KB when stdout is a pipe and the log viewer looks frozen. `CREATE_NO_WINDOW` on Windows. Quoted arguments: `shlex.split(posix=False)` (preserves Windows backslashes) but then **surrounding quotes are stripped per token** — a real bug fix; `C:\my reports\x.csv` must survive round-trip.

### 4.2 Process-tree termination
Killing a `.bat` only kills `cmd.exe` and orphans its children. Stop = psutil walks the process tree, terminates politely, then hard-kills survivors. Keep this — a plain `proc.kill()` regression will look fine in tests and leak processes in production.

### 4.3 Scheduler (`scheduler.py`)
APScheduler wrapper with `coalesce=True`, `max_instances=1`, misfire grace 300s. Frequency→cron translation (in MIS's `schedule_store.py`, same `ScheduleSpec`): `Daily 18:30`→`30 18 * * *`, `Weekly Wed 07:15`→`15 7 * * 2`, `Monthly 5th 06:00`→`0 6 5 * *`. **Scheduled runs only fire while the app is open** — documented limitation; Windows Task Scheduler is the answer for closed-app runs.

### 4.4 One scheduler for the whole app (invariant)
MIS's `ReportSchedule` duck-types exactly what `CronusScheduler` reads (`id`, `name`, `enabled`, `schedule`) and reuses `ScheduleSpec`. A second scheduler means a second set of coalescing/misfire/cron bugs — **do not add one.** Verified: a `ReportSchedule` armed by the unmodified `CronusScheduler` fired correctly; failure paths (missing source, deleted config) are recorded without killing the scheduler thread.

## 5. AI-to-AI Debugging Heuristics

1. **Log viewer "frozen" but script running**: buffering, not threading — confirm `-u`/`PYTHONUNBUFFERED` survived any launcher refactor before touching reader threads.
2. **Orphan processes after Stop**: the psutil tree-kill was bypassed. `.bat` wrappers are the reproducer.
3. **Child receives `"two words"` with literal quotes**: the `shlex.split(posix=False)` + quote-strip combination was altered. Test with a quoted path containing spaces and backslashes.
4. **A schedule fires twice / reports generate twice on double-click**: the UI-side in-flight guard is `self._running: set` keyed by schedule id (Auto-Reporter page); scheduler-side protection is `max_instances=1`. Both exist on purpose.
5. **"Scheduler doesn't run my report at 6 PM"** when the app was closed: not a bug — while-app-open limitation, stated in UI/README/USER_GUIDE. Point the user to Windows Task Scheduler.
6. **Unicode in child output**: ₹ and Bengali verified intact through the streaming path — if garbled after a refactor, check the stream decoding, not the child.
7. Host-app rules apply: engines never import Tkinter; UI callbacks only via `UiDispatcher.post`; GUI has never rendered on a real screen (see the DocuParse doc §5.1 — same risk, same protocol); new deferred imports go into `build.spec` `hiddenimports`.
