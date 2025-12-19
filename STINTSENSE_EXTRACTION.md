# F1 Race Replay → StintSense: Counterfactual Strategy Replay Extraction

This document distills the minimum architecture and logic needed to recreate a **Counterfactual Strategy Replay** UI (e.g., “PIT NOW vs STAY OUT”) without copying code. All references point to the upstream repository for clarity.

---

## 1) Repository Map (what matters for strategy replay)

- **`main.py`** — CLI entry; loads FastF1 sessions, selects race vs qualifying, and launches the appropriate Arcade window.
- **`src/f1_data.py`** — FastF1 data access, multiprocess telemetry extraction, timeline alignment, weather + track status resampling, caching.
- **`src/arcade_replay.py`** — Minimal wrapper that instantiates the race replay window.
- **`src/interfaces/race_replay.py`** — Main render loop (Arcade); track geometry fit/rotation, car drawing, HUD (time/laps/status), interaction handlers, and race progress bar.
- **`src/ui_components.py`** — UI widgets and shared helpers:
  - Track geometry builder and DRS zone detector (`build_track_from_example_lap`, `plotDRSzones`).
  - Progress bar event extraction (`extract_race_events`).
  - Leaderboard, driver info card, weather panel, playback controls.
- **`src/lib/time.py` / `src/lib/tyres.py`** — Utility parsing/formatting and tyre encoding.
- **`src/interfaces/qualifying.py`** — Qualifying-specific view (not core for pit-strategy replay).

---

## 2) Key Modules and Behaviors

### A. Data loading (FastF1) — `src/f1_data.py`
- **Purpose:** Fetch FastF1 session data, extract per-driver telemetry, and serialize to replay-friendly frames.
- **Inputs:** `fastf1.get_session(year, round, session_type)`; CLI flags for cache refresh; optional qualifying session for reference lap.
- **Outputs:** Dictionary with `frames` (time-ordered snapshots), `driver_colors`, `track_statuses`, `total_laps`. Frames include per-driver `x/y`, race distance, lap, tyre (encoded), speed/gear/DRS/throttle/brake, plus optional weather samples.
- **Key mechanics:**
  - **Caching:** Pickle files in `computed_data/`, bypassed with `--refresh-data`. Reuse of precomputed files is assumed safe.
  - **Parallel extraction:** `_process_single_driver` runs per driver (multiprocessing) to collate lap telemetry, convert compound to int, and accumulate race distance.
  - **Timeline alignment:** Builds a common `timeline` at 25 FPS (`FPS` constant) using global min/max session times; resamples all telemetry via `np.interp` onto this grid to produce synchronized frames.
  - **Track status & weather:** Converts `session.track_status` to start/end intervals; resamples weather onto the same timeline.
- **Assumptions:**
  - FastF1 provides complete laps per driver; missing telemetry yields `None`.
  - Leaderboard ordering is by race distance (or later by projected track distance), not official timing gaps.
  - DRS numeric codes (e.g., 8/10/12/14) map to availability/active; tyre compounds converted to fixed ints.
- **Pitfalls:**
  - **Alignment drift:** Pure interpolation on `X/Y` can mis-rank cars around pit entries/exits; pit lane routing isn’t explicitly modeled.
  - **DNF detection:** A driver disappearing from frames is treated as DNF; no explicit retirement flag.
  - **Weather:** If missing or sparse, interpolation may produce `None` values; code is defensive but not gap-aware.
  - **Caching:** Stale cache risk if FastF1 data changes; caller must force refresh.

### B. Time synchronization / driver alignment — `src/f1_data.py`
- **Purpose:** Normalize all driver telemetry onto a shared timebase for consistent playback and ordering.
- **Inputs:** Per-driver `SessionTime` arrays and `X/Y/Distance/RelativeDistance` values from FastF1 laps.
- **Outputs:** Per-frame, per-driver data aligned to the common `timeline` (0-based seconds).
- **Mechanics:**
  - Compute global `t_min`/`t_max` across drivers, then `timeline = np.arange(global_t_min, global_t_max, DT) - global_t_min`.
  - `np.interp` resamples continuous fields; discrete fields (gear) use searchsorted forward-fill; throttle/brake kept as floats (brake scaled to 0–100 in qualifying flow).
  - Leader/lap shown from the frame with greatest race distance; progress bar events derived from sampled frames.
- **Assumptions:** Track distance is monotonically increasing per lap; `RelativeDistance` and `Distance` are sufficient to infer ordering without sectorization.
- **Pitfalls:** Interpolation across pit lane merges can overtake artifacts; start/finish line wrap is handled by lap counter but not by geometric segment snapping.

### C. Track map coordinate transforms — `src/ui_components.py` + `src/interfaces/race_replay.py`
- **Purpose:** Turn telemetry `X/Y` into drawable track geometry and screen coordinates with rotation/scale.
- **Inputs:** Single “example lap” telemetry (prefer qualifying for clean DRS zones) passed from `main.py`.
- **Outputs:** Inner/outer polylines for the track ribbon, DRS zone segments, world bounds, and per-frame car screen positions.
- **Mechanics:**
  - `build_track_from_example_lap`: uses gradients of `X/Y` to compute tangents and normals; offsets by ±(track_width/2) to get inner/outer edges; derives min/max bounds. DRS zones extracted by scanning `DRS` activation states (`plotDRSzones`).
  - `F1RaceReplayWindow.update_scaling`: fits rotated world bounds into the window minus UI margins, preserving aspect ratio; caches scale and translation.
  - `world_to_screen`: optional rotation about track center, then scale/translate.
  - `_project_to_reference`: projects car `x/y` onto a dense reference polyline (4k points) to compute along-track distance for leaderboard ordering.
- **Assumptions:** Example lap has consistent `X/Y/DRS`; fixed track width (200 units) is visually acceptable.
- **Pitfalls:** No smoothing/splines on noisy telemetry; rotation is uniform (no per-sector tweaks); projection chooses nearest polyline point (can jump under overlaps like Monaco tunnel or pit entry).

### D. Event handling (yellow/SC/VSC/red) — `src/f1_data.py` & `src/ui_components.py`
- **Purpose:** Convert FastF1 `track_status` into drawable flag segments and HUD text.
- **Inputs:** `session.track_status` rows with `Status`, `Time`.
- **Outputs:** Structured `{status, start_time, end_time}` array; derived progress-bar events with frame ranges.
- **Mechanics:**
  - `get_race_telemetry`: builds sequential intervals, setting each entry’s `end_time` to the next start.
  - `extract_race_events`: translates status codes (`2`=yellow, `4`=SC, `5`=red, `6/7`=VSC) into events with frame spans; clamps out pre-race artifacts.
  - Render: `on_draw` paints track color per current status; progress bar overlays colored segments.
- **Assumptions:** Track status codes match FIA feed; no overlap between statuses; frame-rate conversion uses fixed 25 FPS.
- **Pitfalls:** If FastF1 emits overlapping or out-of-order statuses, end times may be incorrect; pre/post-race statuses are truncated but not fully filtered.

### E. UI / render loop — `src/interfaces/race_replay.py` + `src/ui_components.py`
- **Purpose:** Drive Arcade window lifecycle, draw track + cars + HUD, and handle interaction.
- **Inputs:** Prebuilt `frames`, `track_statuses`, `example_lap`, `driver_colors`, `total_laps`, window size events, keyboard/mouse.
- **Outputs:** Real-time visualization; user-adjustable playback state (`frame_index`, `playback_speed`, pause); optional progress bar/DRS overlays.
- **Mechanics:**
  - **Frame advance:** `on_update` increments `frame_index` by `delta_time * FPS * playback_speed`; clamps at end.
  - **Draw order:** background → track ribbon → DRS zones → car dots → HUD (lap/time/status) → weather → leaderboard → controls legend → driver info → progress bar → overlays.
  - **Input:** Space/pause, arrow seek, number keys for preset speeds, R to restart, D to toggle DRS zones, B to toggle progress bar. Components expose mouse hit-testing for seeking and selection.
  - **Leaderboard ordering:** uses projected along-track distance to mitigate `Distance` glitches; attaches tyre icon and “OUT” label when `rel_dist == 1`.
- **Assumptions:** 25 FPS cadence is acceptable; single-threaded render loop; textures available under `images/`.
- **Pitfalls:** No rewind buffer beyond frames (cannot scrub before frame 0); projection can still mis-rank during pit lane divergence; UI scaling assumes generous margins and may clip on very small windows.

---

## 3) Reimplement in StintSense — Checklist

**Rebuild from scratch (core to strategy counterfactuals):**
- FastF1 (or equivalent) ingestion with **per-driver timelines**: session load, per-lap telemetry pull, multiprocess optional; normalize to shared FPS; cache invalidation hook.
- **Frame builder** that emits minimal fields needed for strategy replay: time, lap, car pose (`X/Y`), along-track distance, tyre compound, pit state, DRS, speed, gear, throttle/brake (if needed for UI).
- **Event interval model** for flags/SC/VSC and weather samples resampled to the same timeline.
- **Track geometry generator** from a reference lap: tangent/normal offsets, bounds, rotation handling, DRS zone extraction; projection helper for ordering.
- **Render loop skeleton**: background + track ribbon + car markers + basic HUD (time, lap, flag state) + optional progress bar/controls. Keep it deterministic and side-effect free for branching replays.

**Adapt concepts (no code copy):**
- Timeline alignment via interpolation onto a fixed FPS grid.
- DRS zone detection by scanning activation codes on the reference lap.
- Progress bar event lanes (flags, DNF markers) mapped from time intervals to frame ranges.
- Along-track projection to stabilize ordering when `Distance` is noisy.
- UI composability via small components (leaderboard, weather, controls) that read the current frame without global state.

**Must NOT include (out of scope for counterfactual strategy):**
- Fan-facing flourishes: full telemetry dashboard styling, icons, background textures, weather icons, elaborate legends.
- Qualifying modal/segment selector and comparative lap charts.
- Non-strategy features like tyre icon art, throttle/brake bar visuals, and detailed control chrome (unless needed for usability).
- Any copied assets or strings; replicate behavior with StintSense-native UI and assets.

---

## 4) Implementation Notes & Pitfalls to Mitigate

- **Pit lane modeling:** The original ordering assumes a single path; for counterfactuals, treat pit lane as a separate spline with merge logic to avoid jumpy ranks.
- **Branching timelines:** Store strategy branches as immutable frame streams sharing a common timebase so PIT vs STAY can be compared or toggled live.
- **Cache coherence:** Stamp caches with FastF1 version + session metadata; expose manual refresh; validate lap counts before reuse.
- **Status overlaps:** Normalize and merge overlapping flag intervals; enforce start < end and clamp to race bounds.
- **Projection accuracy:** Consider higher-order smoothing (splines) and segment-aware projection (pit vs main) to reduce misordering.
- **Performance:** Precompute track screen coordinates on resize; avoid per-frame heavy math; keep frame objects lightweight for fast branch switching.

---

This summary is designed to guide a clean-room reimplementation of the counterfactual replay layer in StintSense without copying code, while preserving the essential behaviors needed for decision-branch visualization. 
