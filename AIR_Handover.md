# AIR (Automation Intelligence Recorder) - System Architecture & AI Handover Context

Repo: https://github.com/kristic8998/air · Current: **v0.1.0** (beta) · Package name: `air-automation`, import name `air`. CLI entry: `air`.

## 1. Core Purpose & Business Value

AIR turns a *described or recorded* repetitive workflow into two artifacts: (a) a complete, runnable, production-quality **Python automation script**, and (b) a manager-readable **Markdown SOP** describing the same process. The analyst edits a typed workflow model (steps + conditions + loops + error policies), not code. Value: it removes the "I know the process but can't code it robustly" gap — generated scripts come with logging, a typed Config dataclass, retry/continue/stop error policies, and screenshots-on-failure, which hand-rolled scripts never have. It is deliberately a **generator, not a runner**: emitting text is safe, testable, and dependency-light.

## 2. Tech Stack & Dependencies

- Runtime dependency: **only `pydantic>=2.6`** (the whole engine is stdlib + pydantic — keep it that way; this is a core design constraint).
- Optional extras: `run` (pyautogui, selenium, requests, pandas, openpyxl — needed only to *execute generated scripts*), `record` (pynput — for the future Phase-2 live recorder), `dev` (pytest, black, ruff, pyinstaller).
- CI mirrors the house standard (ruff, black, pytest on Ubuntu + Windows). No GUI, so no selftest/display concerns.
- The **generator never imports the automation libraries it emits code for** — tests validate emitted source with `ast.parse`/`compile`, so "generated script is syntactically valid Python" is a hard guarantee, not a hope. Preserve this property in any new emitter.

## 3. Complete Directory Structure

```
air/
├── src/air/
│   ├── __init__.py      # version
│   ├── model.py         # THE single source of truth (see §4.1)
│   ├── generator.py     # Workflow -> Python source text (per-action emitters)
│   ├── templates.py     # 5 built-in starter workflows + list/load registry
│   ├── recorder.py      # CaptureEvent model + events_to_workflow synthesizer
│   ├── store.py         # SQLite Library: versioned workflows + run history
│   ├── docgen.py        # Workflow -> Markdown SOP
│   ├── paths.py         # per-user app-data dir
│   └── cli.py           # `air templates|new|list|show|generate|doc|history…`
├── tests/               # conftest + test_generator + test_model_store_recorder
├── pyproject.toml  README.md  .github/workflows/ci.yml
```

## 4. Architectural Blueprint

### 4.1 Domain model (`model.py`) — one model, four consumers
`Workflow` = ordered `list[Step]` + metadata. Each `Step` = an `ActionType` (enum: open_app, click, double_click, type_text, hotkey, screenshot, wait, open_url, click_selector, fill_selector, download, http_request, excel/file ops…) + typed `params: dict`, optionally wrapped by a `Condition` (run-if), a `Loop` (`LoopKind`: repeat N / for-each), and an `ErrorPolicy` (retry with backoff / continue / stop). `Backend` enum separates desktop (pyautogui) vs web (selenium) vs http vs data actions. **Every other module consumes this one model**: the recorder populates it, templates return it, the generator compiles it, docgen describes it, the store persists it (Pydantic JSON round-trip is lossless — that is what makes SQLite-as-JSON-blobs safe).

### 4.2 Code generator (`generator.py`)
Emits: imports (only for backends actually used), a `Config` dataclass (collected from step params that look configurable), logging setup, a `Workflow` class where **each step becomes its own method** wrapped according to its error policy, a `run()` orchestrating them (conditions/loops composed around step calls), screenshot-on-failure hook, `__main__` guard. Emitters are small pure functions `_emit_<action>(params) -> list[str]`; `_py(value)` safely reprs params into code. To add an action: extend `ActionType`, write `_emit_x`, register it, add a compile-test — nothing else.

### 4.3 Store, recorder, docgen, CLI
- `store.Library` (SQLite, one file under app-data): workflows saved as JSON blobs with **version increment on every save**, plus `RunRecord` history (generated/executed, timestamps, outcome). No server. Schema is embedded as `_SCHEMA` in store.py.
- `recorder.events_to_workflow(events)`: converts a `CaptureEvent` stream into a Workflow, **collapsing noise** (consecutive keystrokes → one `type_text`). The live OS hook (`pynput`) is intentionally NOT implemented — `live_recording_available()` reports capability honestly. Do not fake a live recorder; build it only when it can be tested.
- `docgen.generate_sop(workflow)`: numbered plain-English procedure, tools required, conditions/loops/error handling spelled out, maintenance guide.
- `cli.py`: `air templates`, `air new NAME --template ID`, `air list/show`, `air generate NAME -o script.py`, `air doc NAME -o sop.md`, `air history`.

## 5. AI-to-AI Debugging Heuristics

1. **A generated script has a syntax error** → the bug is in an emitter's string building, never in user data. Reproduce with the failing Workflow JSON → `generator.generate(workflow)` → `ast.parse`. Add that workflow as a regression test. Check `_py()` first (quoting/escaping).
2. **New action type requested** → model enum + emitter + docgen sentence + template usage + compile-test, in that order. If you skip the docgen sentence, SOPs silently say "unknown step" — grep `_step_sentence`.
3. **Pydantic v2 gotchas**: model round-trip uses `model_dump_json`/`model_validate_json`; if a store "loses" fields after an upgrade, a field was added without a default — always give new fields defaults so old library blobs still load.
4. **Do not add runtime deps to the engine.** If a feature needs pandas et al., it belongs in the *generated script* (emit the import), or behind the `run` extra — the generator itself must stay pydantic-only, or the CLI-on-clean-machine story breaks.
5. **SQLite locking on Windows** (rare, if users script around the CLI): Library opens short-lived connections; keep it that way — no long-lived shared connection, no threads writing concurrently.
6. **`events_to_workflow` producing noisy steps** (one step per keystroke): the collapsing rules live there; extend the coalescing window rather than post-processing in the CLI.
7. Version bump touches `pyproject.toml` + `src/air/__init__.py` only (no installer — AIR is a CLI, exempt from the desktop packaging rule).

## 6. Mock Data Simulation & Execution Guide

AIR eats event streams, not spreadsheets, so its simulator fabricates a noisy recorded desktop session.

1. `python mock_data_simulator.py` — repo root (pandas/numpy only). Writes `mock_data/capture_events.json` (47 raw events: 37 per-keystroke `type` events incl. ₹ and Bengali chars, clicks, hotkeys, sub-second waits, an `open_app`) and `events_summary.csv`.
2. Synthesize + generate (from repo root):
   `python -c "import json,sys; sys.path.insert(0,'src'); from air.recorder import CaptureEvent, events_to_workflow; from air.generator import generate_code; ev=[CaptureEvent(**e) for e in json.load(open('mock_data/capture_events.json'))]; print(generate_code(events_to_workflow(ev,'mock-session')))"`
3. Expected results: the 47 events collapse to a **12-step** workflow with exactly **2** `type_text` steps; the printed script must `ast.parse` clean (~186 lines with logging + Config + main guard).
4. Headless validation (2026-07-26): exactly those numbers, verified before publishing. If `type_text` count ≠ 2, the recorder's keystroke coalescing regressed (heuristic §5.6).
