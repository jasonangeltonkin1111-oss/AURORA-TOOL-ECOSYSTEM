# Session Clock Blueprint

## 1) Tool identity
- **Name:** Session Clock
- **Suite:** Session Suite
- **Tiering:** Public core + private-cousin extension hooks
- **Status:** Planned
- **Primary objective:** Make trading-session timing explicit, deterministic, and timezone-safe.

---

## 2) Product definition

### Problem statement
Traders routinely mis-time entries around session opens/closes because local chart time, broker server time, and market-session time diverge (especially during DST transitions). Session Clock provides a single canonical timeline and countdown model.

### Target users
- Intraday FX/index/commodity traders.
- Prop-firm traders with strict session windows.
- Multi-timezone teams sharing setup notes.

### User value
- Know exactly **what session is active now**.
- Know exactly **when next state change occurs**.
- Avoid DST and timezone confusion with deterministic rendering rules.

---

## 3) Exact session map (canonical)

> All session boundaries are represented internally as **IANA timezone + local wall-clock interval(s)** and then converted to absolute instants for runtime countdown.

### 3.1 Named sessions
1. **Sydney** (TZ: `Australia/Sydney`)
2. **Tokyo** (TZ: `Asia/Tokyo`)
3. **London** (TZ: `Europe/London`)
4. **New York** (TZ: `America/New_York`)

### 3.2 Default operating windows (local wall-clock)
- **Sydney:** 08:00–17:00 (Mon–Fri, Sydney local date)
- **Tokyo:** 09:00–18:00 (Mon–Fri, Tokyo local date)
- **London:** 08:00–17:00 (Mon–Fri, London local date)
- **New York:** 08:00–17:00 (Mon–Fri, New York local date)

### 3.3 Optional overlap windows (derived)
- **Tokyo–London overlap:** Intersection of active intervals.
- **London–New York overlap:** Intersection of active intervals.
- Overlaps are computed from converted absolute instants, never hard-coded by UTC hour.

### 3.4 Weekend/closure behavior
- Session intervals are generated only for valid local weekdays.
- If a generated interval lands on a market-closed day (from calendar provider), mark as `suppressed_by_holiday` and exclude from active-state decisions.

### 3.5 Session map precedence
1. Exchange/broker holiday calendar suppressions.
2. Session local-time definitions.
3. User display timezone transformation.

---

## 4) Timezone behavior

### 4.1 Canonical time model
- Store definitions as:
  - `session_id`
  - `iana_timezone`
  - `start_local_time`
  - `end_local_time`
  - `weekday_mask`
- Runtime computes `start_instant_utc` / `end_instant_utc` for each interval occurrence.

### 4.2 Display timezone modes
- **Mode A: Session-local display** (each row shown in its own session timezone).
- **Mode B: User-local display** (all sessions rendered in user-selected IANA timezone).
- **Mode C: Broker-server display** (all sessions rendered in broker server timezone).

### 4.3 Required timezone rules
- Never use fixed UTC offsets for named sessions.
- Always use IANA timezone conversion APIs with DST-aware rules.
- Display label must include abbreviation and offset for current instant (e.g., `EDT (UTC-4)`).
- If timezone database is unavailable/stale, enter degraded mode and freeze countdown updates with visible warning.

### 4.4 DST transition handling
- On spring-forward day (missing local hour), any interval crossing missing time is normalized to the next valid instant.
- On fall-back day (repeated local hour), ambiguous local times must resolve using explicit policy:
  - `earlier` for session start.
  - `later` for session end.
- Policy must be stable and documented in telemetry payload.

---

## 5) Countdown logic (deterministic)

### 5.1 State machine
`PRE_OPEN -> OPEN -> PRE_CLOSE -> CLOSED`

### 5.2 Timing thresholds
- `PRE_OPEN`: from `next_open - pre_open_window` until `next_open`.
- `OPEN`: from `open` inclusive to `close` exclusive.
- `PRE_CLOSE`: from `close - pre_close_window` until `close`.
- `CLOSED`: otherwise.

Default windows:
- `pre_open_window = 00:30:00`
- `pre_close_window = 00:15:00`

### 5.3 Countdown target selection
Priority per session tile:
1. If `OPEN` and not in pre-close window -> target `close`.
2. If `PRE_CLOSE` -> target `close`.
3. If `PRE_OPEN` -> target `open`.
4. If `CLOSED` -> target `next_open`.

### 5.4 Update cadence
- UI tick every 1 second.
- Recompute schedule anchors every minute and at any timezone/day boundary change.
- On app resume from background, force full recomputation before first repaint.

### 5.5 Negative/invalid countdown protection
- Clamp displayed countdown to `00:00:00` when target passed but state not yet recomputed.
- If target is invalid/missing, show `--:--:--` + `schedule_unavailable` badge.

---

## 6) UI states and behavior

### 6.1 Global panel states
- `loading_schedule`
- `ready`
- `degraded_time_service`
- `market_closed_global`
- `error`

### 6.2 Session tile states
- `open`
- `pre_open`
- `pre_close`
- `closed`
- `suppressed_by_holiday`
- `unknown`

### 6.3 Required UI fields per tile
- Session name
- Current state chip
- Local hours label (source timezone)
- Rendered hours label (active display mode timezone)
- Countdown label (`to open` / `to close`)
- DST indicator when transition occurs within next 72 hours

### 6.4 Interaction rules
- User may switch display timezone without mutating canonical session map.
- Switching timezone triggers immediate re-render + silent schedule recomputation.
- Tooltip must explain if displayed time differs from session-local time.

---

## 7) Acceptance criteria (including DST/timezone edge cases)

### 7.1 Core correctness
- Given any timestamp, active session state is identical across repeated evaluations.
- Session overlap badges match interval intersection computed from UTC instants.
- Countdown never increases by more than one tick between consecutive seconds (excluding manual clock adjustments).

### 7.2 DST-specific acceptance tests
1. **US spring-forward case** (`America/New_York`, second Sunday in March):
   - Session transitions still occur at 08:00/17:00 New York local wall-clock.
   - UTC equivalents shift automatically after DST change.
2. **UK spring-forward case** (`Europe/London`, last Sunday in March):
   - London session remains 08:00/17:00 London local, with UTC shift reflected.
3. **Asynchronous DST gap period (US vs UK mismatch weeks):**
   - London–New York overlap duration adjusts dynamically; no hard-coded overlap hours.
4. **Fall-back ambiguity** (US first Sunday in November / UK last Sunday in October):
   - Ambiguous times resolve with `start=earlier`, `end=later` policy.
   - No duplicate open event emitted.
5. **Non-DST zone control** (`Asia/Tokyo`):
   - Tokyo session UTC mapping remains stable across all DST transitions in other zones.

### 7.3 Timezone-display acceptance tests
- Switching from user timezone A to B changes only rendered labels/countdowns, not canonical session definition.
- Broker timezone mode matches broker server clock offset at current instant.
- If timezone DB unavailable, warning banner appears and countdown stops within 2 seconds.

### 7.4 Boundary/edge acceptance tests
- Exactly at open instant: state = `OPEN`, countdown target = close.
- Exactly at close instant: state = `CLOSED` unless next session immediately opens.
- Weekend/holiday suppression: session tile shows `suppressed_by_holiday` with no actionable countdown.

---

## 8) Out of scope (explicit)
- Forecasting volatility/liquidity from session timing.
- Auto-execution or order routing based on session state.
- Venue-specific micro-session modeling (auction phases, lunch breaks) in public core.
- Historical backtest analytics of session performance.
- Custom user-defined arbitrary sessions in public core.

---

## 9) Private-cousin extension hooks

> Public blueprint intentionally leaves extension interfaces without exposing private strategy logic.

### 9.1 Hook surface
- `onSessionStateChanged(event)`
- `onPreOpenWindowEntered(event)`
- `onPreCloseWindowEntered(event)`
- `onTimezoneModeChanged(event)`
- `resolveCustomCalendar(session_id, date)`

### 9.2 Event contract (minimum)
- `event_id` (uuid)
- `session_id`
- `previous_state`
- `new_state`
- `event_instant_utc`
- `render_timezone`
- `dst_context` (`none|transition_in_24h|transition_now`)

### 9.3 Extension constraints
- Hooks must be non-blocking and time-boxed (<50 ms per callback).
- Hook failures cannot alter baseline session-clock state decisions.
- Private modules may subscribe but cannot override canonical timezone/DST policies.

---

## 10) Telemetry and QA checklist

### 10.1 Telemetry events
- `session_clock_rendered`
- `session_state_changed`
- `countdown_target_changed`
- `timezone_mode_changed`
- `dst_transition_detected`
- `time_service_degraded`

### 10.2 QA checklist
- Validate canonical intervals across one full year for each session timezone.
- Run DST boundary replay tests for US/UK/AU transitions.
- Verify no duplicate or skipped state transitions under clock drift correction.
- Verify degraded mode UX when timezone service fails.
- Verify private hook failures are isolated and logged.

---

## 11) Release metadata
- **Version:** `1.0.0-session-clock-blueprint`
- **Owner:** Session Suite
- **Change note:** Replaced generic blueprint with deterministic session map, timezone/DST-safe runtime model, explicit UI/state machine, strict acceptance criteria, out-of-scope list, and private-cousin extension hooks.
