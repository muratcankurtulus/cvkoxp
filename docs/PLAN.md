# cvkoxp — project charter and modernization plan

> Status: approved plan, not yet implemented. Written on macOS; execution begins on the Windows
> development machine that hosts the game client and the private server.

## 1. Project context

**What this system is for.** `cvkoxp` is a black-box QA harness for a **privately operated,
self-hosted Knight Online server emulator**. The operator configures drop tables server-side and
needs to verify empirically that the configured rates are what the client actually observes. The
harness automates a character through a farming loop, reads item-drop messages from the client's
chat log, and produces drop-rate statistics with confidence intervals to compare against the
configured values.

**Operating environment — all first-party:**

- The **server** is private infrastructure owned and administered by the operator.
- The **client** runs on the operator's own Windows development machine.
- **No other players exist on this server.** It is a single-operator test environment, not a live
  service.
- There is no third party to disadvantage, no live-service terms being circumvented, and no
  competitive context.

**Why input automation is required.** The measurement is deliberately black-box: the harness reads
what a *player* sees rather than querying the server database, because the thing under test is the
whole stack — server drop-table config, network layer, and client presentation together. A DB read
would bypass most of what is being verified. Producing a statistically meaningful drop-rate estimate
needs thousands of kills, which is not feasible by hand. This is the same category of work as
Selenium or Playwright driving a browser, or an HID emulator driving a device under test: synthetic
input against a first-party system to make a test loop repeatable.

**Why the project is being reworked now.** Two things changed at once:

1. **The existing AutoHotkey input path stopped working** — keys no longer register in the client.
   This is the hard blocker; every other capability is downstream of being able to drive the
   character.
2. **The purpose changed.** `cvkoxp` began years ago as a hobby autofarm script. It is now a
   measurement instrument. The farming loop is no longer the product — it is the *data collection
   mechanism*. The product is a trustworthy estimate of observed drop rates.

That second point drives every design decision below. A farming loop that works 95% of the time is
fine. A *measuring instrument* that miscounts 5% of drops is worse than useless, because it produces
confident wrong numbers. The plan is organized around making the measurement trustworthy and,
crucially, around making its error rate **visible**.

**Scope boundary, stated up front and unchanged by any of the above:** this plan does not include
circumventing anti-cheat or client protection software. Phase 0 diagnoses *why* input injection
stopped working. If the answer turns out to be a fixable API-level mismatch — wrong integrity level,
virtual-key vs scancode, an AutoHotkey version trap — that gets fixed. If the answer is that a
protection layer is actively rejecting injected input, that is where the work stops and the operator
decides how to proceed. The diagnostic tells the truth either way; that is its whole point.

## 2. Development environment constraint

Development happens on **macOS**. The game client and the private server run on a separate
**Windows** machine.

This is a genuine architectural driver, not an inconvenience. The core — perception, policy,
analytics, store, API — must import, run, and be fully tested on macOS against **recorded frames**.
Only screen capture and input injection are Windows-only, isolated behind Protocols and installed
via an optional `[windows]` dependency group.

The record/replay harness is therefore load-bearing infrastructure, not a testing nicety: it is the
only way the majority of this codebase can be developed at all.

## 3. Current state

`bot.py` is 257 lines in a single file: one `Agent` god-class, four raw threads coordinated by
closure-over-boolean flags, magic screen coordinates for 1920x1080, a hardcoded Tesseract path,
Tesseract OCR for everything including HP/MP numerals, and `difflib` fuzzy matching for target
names. No tests, no config, no packaging, no type checking, no logging. The module is not importable
— `bot` is a global assigned inside `if __name__ == '__main__'`.

It works (or worked). It is not a base to build a measuring instrument on.

## 4. Decisions already taken

Recorded so they are not relitigated later:

- **Drop ground-truth: client-side chat-log OCR only.** Black-box by intent (see §1). No server DB
  integration.
- **Expected rates: declared by hand** in `experiments/drop-rates.toml`.
- **Run model: continuous session logging.** Farm indefinitely, stream to the datastore, slice
  afterward in the dashboard. No declarative pass/fail experiment runner.
- **Control surface: Textual TUI + FastAPI/React web dashboard.**
- **Perception: classical CV + modern OCR.** No trained detection model in this plan.

---

## Phase 0 — `cvkoxp doctor`: diagnose the input breakage

**Ship this first, standalone, before any refactoring.** Nothing else is worth building until keys
register, and the answer may change which input backend the rest of the plan targets.

`doctor` runs an ordered diagnostic and prints a verdict, replacing guesswork with measurement:

1. Process architecture / OS sanity; DPI-awareness mode (silently affects every coordinate in the
   system).
2. **Integrity-level comparison** between the harness process and the game process
   (`GetTokenInformation` / `TokenIntegrityLevel`). A mismatch means UIPI discards injected input
   with no error surfaced anywhere — the most common cause of "it just stopped working one day."
   The current code self-elevates for exactly this reason (`bot.py:254`) but never verifies the
   elevation succeeded or that the levels actually match.
3. **Window discovery:** enumerate all top-level windows and report every title fuzzy-matching
   "Knight". Current code hard-matches the exact string `Knight OnLine Client` (`bot.py:18`) and
   fails silently if the client's title changed — plausible across client versions.
4. **Foreground acquisition:** call `SetForegroundWindow`, then verify with `GetForegroundWindow`
   rather than trusting the return value, which lies under several documented conditions.
5. **AutoHotkey probe:** is `AutoHotkey.exe` on PATH, and **what major version does it report?** The
   `ahk` PyPI package shells out to that binary. AHK v1 (`AutoHotkey_L` — what the README currently
   links) is end-of-life, and a v2 installation silently fails a v1-syntax wrapper. **Leading
   hypothesis for the breakage**, given the age of the repo.
6. **Round-trip injection test:** install a `WH_KEYBOARD_LL` hook, fire a key through each backend,
   and report (a) whether the event reaches the OS at all and (b) whether `LLKHF_INJECTED` is set on
   it.
7. **Scancode vs virtual-key test:** fire both variants, report which the OS observes. Many game
   clients read scancodes via DirectInput/Raw Input and ignore virtual-key-only injection; the fix
   is `KEYEVENTF_SCANCODE` with `MAPVK_VK_TO_VSC` translation. **Second leading hypothesis.**

Deliverable: `uv run cvkoxp doctor` emits a ranked list of what is broken and which backend to use.

**Known constraint, flagged not solved:** `SendInput` targets the foreground window, so the client
must hold focus for the duration of a run — the machine is unusable while farming. A VM or a
dedicated box is the practical answer for multi-hour sessions.

## Phase 1 — Foundation

Standard 2026 Python toolchain, `src/` layout, single-file configuration in `pyproject.toml`.

- **uv** (packaging/venv/lock), **ruff** (lint + format), **ty** for fast local type checking with
  `mypy --strict` as the authoritative CI gate until ty leaves beta, **pytest** +
  `pytest-asyncio` + `hypothesis` (property tests fit the statistics layer well). Target Python
  3.13+.
- **pydantic v2** + `pydantic-settings` for all configuration. **structlog** for JSON-lines
  structured logging the dashboard can consume directly.
- Optional dependency groups: `[windows]` (capture + input), `[web]`, `[dev]`. **The default install
  must work on macOS.**
- Repo guardrails per the operator's standard workflow: pre-commit (ruff, gitleaks, commit-msg
  hook), GitHub Actions for CI + secret scanning + Conventional Commits + release-please.

## Phase 2 — Capture and input behind Protocols

The layer that makes cross-platform development possible.

**`capture/base.py`** — `CaptureBackend` Protocol returning frames plus a monotonic timestamp.

- `wgc.py` — Windows Graphics Capture (via `windows-capture`). Primary; handles more window states
  than DXGI.
- `dxgi.py` — `dxcam` / Desktop Duplication. Fallback.
- `mss_backend.py` — cross-platform, for development on macOS.
- `replay.py` — **replays a recorded frame directory.** This is what lets the entire system run
  headless in CI with no game attached.

> **Gotcha:** `.gitignore` currently has a blanket `*.png`, which will silently swallow recorded
> frame fixtures under `tests/fixtures/frames/`. Those fixtures must be committed or the CI replay
> regression has nothing to run against. Add a negation (`!tests/fixtures/frames/**/*.png`) when
> that directory is created, or store fixtures as `.npz`.

Capture only the configured ROIs, never the full screen. `ImageGrab.grab` on every frame plus a full
OCR pipeline is what makes the current version expensive; a QA harness runs for hours, so idle cost
matters far more than peak framerate. 10–20 Hz is ample.

**`input/base.py`** — `InputBackend` Protocol (`key_press`, `key_down`, `key_up`, `mouse_move`,
`mouse_click`).

- `sendinput.py` — **in-process ctypes `SendInput`, scancode-based.** Primary. Eliminates the
  AutoHotkey subprocess, the PATH dependency, and the v1/v2 version trap in a single move.
- `ahk.py` — the existing AutoHotkey path, v2-aware. Kept for comparison and fallback.
- `null.py` — logs intents, sends nothing. **Default**, and what tests and CI use.

## Phase 3 — Perception

**`perception/regions.py`** — named ROIs resolved from a layout profile
(`profiles/1920x1080-scale1.3.toml`), replacing magic numbers scattered through `bot.py:57-64`,
`bot.py:89-91`, `bot.py:109`. A `cvkoxp calibrate` command drags ROIs on a screenshot and writes the
profile — this is what unlocks resolutions other than 1920x1080.

**`perception/bars.py`** — HP/MP by **pixel-ratio analysis, not OCR**. Count filled pixels along a
scanline within the bar's hue range. Microseconds instead of ~80ms, and effectively deterministic.
OCR'ing `245/580` at 12px (`bot.py:87-106`) is simultaneously the slowest and the most failure-prone
thing the current code does.

**`perception/ocr.py`** — `OcrEngine` Protocol; **RapidOCR** (ONNX runtime) as the implementation,
removing the hardcoded Tesseract binary path (`bot.py:23`) and the system-level install entirely.
Preprocessing: crop → Lanczos upscale → CLAHE → adaptive threshold. Single-line ROIs use single-line
page segmentation rather than the current `--psm 3` (`bot.py:75`), which is the wrong mode for a
one-line banner.

**`perception/chatlog.py`** — the most important and most subtle module in the system, because every
drop statistic flows through it.

The chat log **scrolls**. Naive per-frame OCR both double-counts (a line stays visible across many
frames) and misses lines (scrolled past between two captures). Both failure modes silently corrupt
precisely the number being measured. Approach:

- Segment the ROI into lines via horizontal projection profile; OCR per line.
- Track inter-frame scroll offset by phase correlation, so a genuine repeat of an identical message
  is distinguishable from the same line re-rendered one row higher.
- Deduplicate on normalized-text hash within a rolling window keyed by scroll position.
- **Match against a closed dictionary.** The item names are already known — they come from the
  expected-rates TOML. `rapidfuzz.process.extractOne` against that dictionary with a score cutoff is
  dramatically more reliable than free-form OCR, and it is the correct fix for the README's "can't
  read the name against busy backgrounds" TODO.
- **Quarantine every unmatched line** into its own table. Non-negotiable for a measuring instrument:
  unmatched lines *are* the miss rate. A miss rate you can see is a caveat; a miss rate you cannot
  see is a lie.

Target-name matching also moves to `rapidfuzz`, replacing `difflib.SequenceMatcher` (`bot.py:81`),
with a **rolling vote across N frames** instead of trusting a single read.

**Change-gated OCR:** hash each ROI's pixels and skip OCR when unchanged. The chat log changes once
or twice a second, so this skips the large majority of OCR calls.

**`perception/observation.py`** — a frozen dataclass holding one tick's world state. This is the
critical seam: policy consumes `Observation` and nothing else, so the entire decision layer is
testable with hand-written fixtures and no screen at all.

## Phase 4 — Policy and runtime

**`policy/state.py`** — explicit state machine over an `Enum` with typed transitions, replacing the
stringly-typed `self.state` (`'init'` / `'onhold'` / `'toofar'` / `'attacking'`) threaded through the
current `Agent`.

**`policy/scheduler.py`** — per-skill `cast_time`, `cooldown`, `priority`. The scheduler refuses to
issue a new intent while a cast window is open. **This is the fix for the README's long-cast-time
TODO** — `wolf` currently fails because `reccurent_skills` fires on a blind 121-second timer with no
knowledge of what else is casting (`bot.py:239`).

**`policy/intent.py`** — policy emits `Intent` objects (`PressKey`, `Wait`, `Retarget`); it never
touches input directly. This makes the decision layer a pure function of
`Observation → list[Intent]`, and therefore trivially unit-testable.

**`runtime/loop.py`** — **one asyncio loop at a fixed tick rate**, capture in a thread executor,
replacing the four raw threads coordinated by closure-over-boolean flags (`bot.py:212-253`).
Cancellation via `TaskGroup` / `CancelledError`. Errors get bounded exponential backoff instead of a
bare `except: print(e)` inside a tight loop.

## Phase 5 — Event store and analytics

**`store/`** — append-only SQLite event log (WAL mode), plain `sqlite3` behind a thin repository.
Events: kills, drops, quarantined lines, state transitions, HP/MP samples, OCR confidences.
Append-only means any session remains re-analyzable without re-farming it.

**`analytics/`** — the actual deliverable of the project.

- **Wilson score intervals**, not the normal approximation. Drop rates are low-probability events
  where the normal approximation is badly wrong at realistic sample sizes.
- **Binomial test** per item against the hand-declared expected rate in
  `experiments/drop-rates.toml`.
- **Chi-square goodness-of-fit** across the whole drop table.
- **Sample-size calculator:** given an expected rate and a target precision, how many kills are
  required. Directly answers "how long do I need to leave this running," which is the question that
  will actually get asked.
- **Instrument error reporting:** quarantine rate and OCR confidence distribution surfaced
  *alongside every rate estimate*, so an observed rate is never displayed without its measurement
  uncertainty.

## Phase 6 — Control surfaces

**`tui/app.py`** — Textual dashboard: start/stop, live state, kills/hr, running rate estimates,
error counts. Zero-friction terminal control on the Windows box.

**`api/`** — FastAPI; SSE for live event streaming, REST for historical queries against the store.

**`web/`** — Vite + React + TypeScript. Its highest-value screen is not the charts — it is the
**live annotated frame view**: the current capture with ROI overlays, per-region OCR reads, and
confidences drawn on top. The hardest part of any CV pipeline is seeing what it sees, and this is
where the full-stack effort pays for itself. Plus rate tables with confidence intervals,
observed-vs-expected charts, and a quarantine inspector for reviewing unmatched lines.

---

## Concrete defects being fixed

All of these disappear in the rewrite; listed so none is silently lost.

| Location | Defect |
|---|---|
| `bot.py:2` | `from re import T` — dead import |
| `bot.py:18` | Hard-matched window title; silent failure if it changed |
| `bot.py:23` | Hardcoded Tesseract binary path |
| `bot.py:58-64` | `window_info['width']/['height']` are sizes but passed to `get_screen` as bottom-right coords; `x1` is reassigned before `x2` is derived from it |
| `bot.py:68` | Writes `image.png` to disk on every frame, inside the hot loop |
| `bot.py:80` | `assert` used for type validation — stripped entirely under `-O` |
| `bot.py:104-106` | `text[1]` indexed with no length check; `IndexError` whenever OCR returns no `/` |
| `bot.py:120-125` | `check_toofar` — both branches call `set_target()`; state oscillates between `toofar` and `onhold` |
| `bot.py:137-140` | `start_attack` blocks 2.4s inside the control loop |
| `bot.py:170-180` | Bare `except: print(e)` with no sleep on the error path — 100% CPU spin plus log flood |
| `bot.py:192-205` | Tight keypress loop with no sleep when `t is None` |
| `bot.py:182-190` | `check_target` thread lacks the foreground-window guard the other two workers have |
| `bot.py:210` | `bot` is a `__main__`-only global — module not importable, nothing testable |
| `bot.py:225-229` | `att stop` before `att start` raises `NameError` |
| `bot.py:214` | Turkish prompt `Aksiyon Gir:` in an otherwise English codebase |
| — | No cooldown / cast-time model at all |
| — | Non-daemon threads with no join timeout — Ctrl-C does not exit cleanly |

## Verification

**On macOS (no game client required) — the everyday loop:**

```bash
uv run ruff check && uv run ty check && uv run mypy --strict src/
uv run pytest                                   # full core suite vs recorded fixtures
uv run cvkoxp replay tests/fixtures/frames/session-01 \
    --assert tests/fixtures/observations.jsonl  # deterministic perception regression
uv run cvkoxp report --ocr-accuracy             # vs hand-labeled fixtures; CI-gated threshold
```

**On the Windows machine:**

```bash
uv run cvkoxp doctor                            # Phase 0 — which input path works, and why
uv run cvkoxp record --duration 300             # capture fixtures; copy to mac for the suite above
uv run cvkoxp run --profile 1920x1080-scale1.3 --dry-run   # full loop, NullInput, client untouched
uv run cvkoxp run --profile 1920x1080-scale1.3             # live
```

**End-to-end acceptance for the measurement itself:** farm a monster with a known-high-rate item,
screen-record the session, hand-count the drops, and compare against the harness's count. The
harness becomes trustworthy when its miss rate is **measured**, not when it merely looks correct.

## Delivery

One worktree and one short-lived branch per phase, Conventional Commits, squash-merge,
release-please for versioning.

`fix/input-doctor` → `chore/tooling-foundation` → `feat/capture-input-backends` →
`feat/perception` → `feat/policy-runtime` → `feat/store-analytics` → `feat/tui` →
`feat/web-dashboard`

Phase 0 ships first and is independently useful — it answers the question currently blocking
everything else, regardless of whether the rest of the plan proceeds.

## Non-goals

- **No anti-cheat or client-protection circumvention.** Phase 0 diagnoses the input failure. Fixable
  API-level mismatches get fixed; an active protection layer rejecting injected input is where the
  work stops.
- **No server-side integration.** Black-box client-side measurement only — that is the point of the
  test, not a limitation (see §1).
- **No trained detection model.** The record/replay harness produces a labeled dataset nearly for
  free, so this stays a clean follow-on if dictionary-constrained OCR proves insufficient.

## Reference

Stack choices researched August 2026:

- [uv + Ruff + ty — the 2026 Python toolchain](https://www.kdnuggets.com/python-project-setup-2026-uv-ruff-ty-polars)
- [Python OCR library comparison](https://www.codesota.com/ocr/best-for-python) — RapidOCR vs Tesseract vs PaddleOCR
- [DXcam](https://github.com/ra1nty/DXcam) — Desktop Duplication capture
- [RapidFuzz vs difflib](https://github.com/rapidfuzz/RapidFuzz/blob/main/api_differences.md)
- [Textual](https://www.textualize.io/blog/7-things-ive-learned-building-a-modern-tui-framework/)
