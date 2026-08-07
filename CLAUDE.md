# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project context

`cvkoxp` is a **black-box QA harness for a privately operated, self-hosted Knight Online server emulator**. The operator configures drop tables server-side and uses this to verify empirically that the configured rates are what the client actually observes.

Everything in the loop is first-party: the server is private infrastructure the operator administers, the client runs on the operator's own Windows dev machine, and **no other players exist on the server** — it is a single-operator test environment, not a live service.

The measurement is deliberately black-box. It reads what a *player* sees rather than querying the server database, because the thing under test is the whole stack — drop-table config, network layer, and client presentation together. Automated input is what makes the loop repeatable at the sample sizes a rate estimate needs; it is the same category of work as Playwright driving a browser or an HID emulator driving a device under test.

The project began years ago as a hobby autofarm script, and `bot.py` still reflects that. It is being reworked because the purpose changed: **the farming loop is now the data-collection mechanism, not the product.** The product is a trustworthy drop-rate estimate. That distinction drives the design — a farming loop that works 95% of the time is fine, but a measuring instrument that miscounts 5% of drops produces confident wrong numbers.

**Read `docs/PLAN.md` before making changes.** It is the approved charter and modernization plan: architecture, phase ordering, decisions already taken (and why), and explicit non-goals.

## Current status

`bot.py` is the original implementation and remains the reference until Phase 1 of the plan begins. **The AutoHotkey input path no longer works** — keys do not register in the client, which blocks everything downstream. Phase 0 of the plan (`cvkoxp doctor`) is a diagnostic for that failure and ships first. Leading hypotheses, in order: an AHK v1/v2 version mismatch (the `ahk` package shells out to `AutoHotkey.exe`, and the v1 the README links is EOL), virtual-key vs scancode injection, and a UIPI integrity-level mismatch.

Scope boundary that applies to all work in this repo: **no anti-cheat or client-protection circumvention.** Fixable API-level mismatches get fixed; if a protection layer is actively rejecting injected input, that is where the work stops.

## What is here today

A hobby single-file screen-scraping script. It reads the game window with OCR and drives it by sending keystrokes via AutoHotkey. There is no build system, no test suite, no linter config, and no package structure — the entire program is `bot.py`. The rest of this document describes that file as it currently stands.

## Running

Windows-only. Requires Python 3.9, Tesseract-OCR, and AutoHotkey both on `PATH`.

```
pip install -r requirements.txt
python bot.py
```

The script self-elevates: if not running as admin it re-launches itself through `ShellExecuteW(..., "runas", ...)` and the original process exits without starting the bot (`bot.py:254`). Elevation is required because the game window runs elevated and `SetForegroundWindow`/key injection fails otherwise.

`requirements.txt` is incomplete — `pywin32` (for `win32gui`) and `numpy` are imported but unlisted. Add them if touching dependency setup.

The Tesseract binary path is hardcoded to `C:\Program Files\Tesseract-OCR\tesseract.exe` (`bot.py:23`).

## Runtime environment assumptions

These are load-bearing; changing them breaks OCR silently (bad reads, not exceptions):

- 1920x1080 screen resolution.
- In-game UI scale set to 1.3 (Options > Game Option1).
- Game window titled `Knight OnLine Client`.

## Architecture

**`Agent` (`bot.py:14`)** — the whole bot. Holds a `state` string that is the de facto state machine:

- `init` → no target yet
- `onhold` → target lost / name mismatch, needs re-targeting
- `toofar` → game printed "too far" in the message log
- `attacking` → target confirmed, attack loop active

`set_target()` presses `z` (game's nearest-target key) and, if a target was confirmed, calls `start_attack()`, which spams `9` three times (approach/opener) and flips state to `attacking`.

**Perception is all screen OCR.** Every check follows the same pipeline: `ImageGrab.grab(box)` → numpy array → `cv2.resize` → grayscale → OTSU threshold → `pytesseract.image_to_string`. Three separate readers:

- `check_target()` (`bot.py:56`) — crops the target-name banner using the window rect from `win32gui`, thresholds both normal and inverted (the name renders light-on-dark or dark-on-light depending on background), and fuzzy-matches against the user-supplied monster name with `SequenceMatcher` at a 0.5 ratio. Anything below that sets state to `onhold`.
- `check_stats(stat_name)` (`bot.py:87`) — reads the HP/MP bars at **hardcoded absolute screen coords** (hp `124,49,196,61`; mp `124,71,196,81`) and parses `current/max` with a regex.
- `check_toofar()` (`bot.py:108`) — reads the chat/message region at hardcoded `1532,955,1871,977` looking for the string "too far".

Note the coordinate inconsistency: `check_target` derives its box from `win32gui.GetWindowRect`, where `window_info['width']/['height']` are sizes, not bottom-right coords — but `get_screen` takes an absolute `(x1,y1,x2,y2)` box. The stat and message readers ignore the window entirely and use fixed full-screen coordinates. Anything that touches these crops needs to be verified visually; `check_target` dumps its crop to `image.png` for exactly that (gitignored).

**Actuation is all keypresses** through `ahk.key_press`. The key map is a hard convention shared with the player's in-game skill bar:

| Key | Meaning |
| --- | --- |
| `z` | select nearest target |
| `9` | approach / attack opener |
| `1` | HP potion (fired below 35% HP) |
| `2` | MP potion (fired below 10% MP) |
| `6` | slot for "light feet" / "safety" |
| `7` | slot used by the `safety` command |
| `8` | slot for "strength of wolf" |
| user-supplied | main attack skill number, prompted at `att start` |

The README documents `6` for both light-feet and safety, but the code binds `safety` to `7` (`bot.py:246`). Trust the code.

**Concurrency.** The REPL in `__main__` (`bot.py:212`) spawns one daemon-less `Thread` per concern and stops them via **closure-over-boolean** flags: each `*_start` sets `x_stop = False` and passes `lambda: x_stop` into the worker, which polls it each iteration. Consequences to be aware of when editing:

- `bot` is a module-level global assigned inside `if __name__ == '__main__'`, and the thread bodies (`cont_attack`, `check_target`, `reccurent_skills`) reference it. The module is not importable as a library.
- Issuing `att stop` before `att start` raises `NameError` — the thread handles are locals created only on start.
- The worker loops wrap their body in bare `try/except Exception: print(e)` and keep spinning, so OCR/window failures show up as a scrolling console log rather than a crash.
- `cont_attack` and `reccurent_skills` no-op unless the foreground window title contains `Knight OnLine Client`, so alt-tabbing pauses the bot. `check_target` does **not** have this guard and keeps grabbing the screen regardless.
- `reccurent_skills` with no `t` argument presses its key in a tight loop with no sleep (used for `lf`/`safety`); `wolf` passes `t=121` for a 121s buff re-cast cycle.

## Console commands

The prompt is `Aksiyon Gir:` (Turkish, "enter action"). Recognized strings — exact match, no parsing:
`att start`, `att stop`, `lf start`, `lf stop`, `wolf start`, `wolf stop`, `safety start`, `safety stop`. `att start` then prompts for the monster name and the attack skill number.

## Known limitations (from README TODO)

- Target-name OCR fails against busy backgrounds.
- No handling for long cast times — `wolf` in particular can be interrupted by the attack loop.
- No GUI.
