# Appliance Smart Control

A Home Assistant **blueprint** that pauses long-running appliances
(dishwasher, washing machine, tumble dryer) shortly before sunset when
household solar export drops below a threshold, so the rest of the cycle
runs on free solar the next day instead of grid power.

Built around **LG ThinQ** entities but works with any appliance that
exposes a status sensor and either an operation `select` or a power
`switch`.

---

## How it works (plain English)

Picture a washing machine that started at 16:30 and still has 90 minutes
left. Sunset is at 17:45. Without help, the second half of the cycle
runs on grid power after dark. This blueprint watches for that and, if
your solar is already fading, hits "pause" before sunset and resumes
the cycle the next morning when the sun is back.

### The basic idea

The automation re-evaluates whenever something interesting happens —
the appliance's state changes, grid export swings, the sun sets, or a
5-minute safety-net tick fires. On each evaluation it asks:

1. **Is the appliance actually running?** (looks at the status sensor)
2. **Are we close to sunset?** (within `lead_minutes`, default 60)
3. **Is solar export already low?** (below `export_threshold`, default
   300 W — meaning we're not exporting much, so the appliance is mostly
   pulling from the grid)
4. **Has export been low for a while?** (`pause_dwell` minutes — so a
   passing cloud doesn't trigger a pause)
5. **Is there enough cycle left to bother?** (between
   `min_remaining_minutes` and `max_remaining_minutes` — skip if it's
   nearly done OR has barely started)
6. **Is it smart to pause right now?** A series of "actually, never
   mind" gates (all optional):
   - Will today's remaining solar finish the cycle anyway? → don't pause
   - Is tomorrow forecast to be cloudy/rainy? → don't pause (no point
     deferring to a bad solar day)
   - Is the grid currently cheap? → don't pause (cheap grid beats
     waiting)
   - Does the home battery have plenty of charge? → don't pause (the
     battery will cover the appliance)

If all of that says "yes, pause it" — it does. It also flips an
`input_boolean` flag to remember **"I paused this, the user didn't"**,
so it never tries to auto-resume something you paused yourself.

### The next day: resuming

You pick one of three resume strategies:

- **`auto_export`** — the next morning, once solar export is back above
  the threshold and stays there for `resume_dwell` minutes, resume
  automatically.
- **`fixed_time`** — resume at a fixed time-of-day you've configured
  (e.g. 09:00).
- **`manual`** — just notify; you press resume yourself.

Two safety rails apply to all auto-resume strategies:

- It won't resume before `earliest_resume_hour` (default 08:00) — no
  starting the dryer at 04:00 because of a sensor glitch.
- It won't resume within `min_pause_hours` of the pause (default 4 h) —
  no instant "pause then immediately resume" loops.
- It won't resume twice on the same day (if you've configured the
  optional `last_resume_date` helper).

### What about appliances I can't control?

For the dishwasher (LG ThinQ exposes no control entities), use
`pause_mode: notify_only`. The blueprint just sends a notification
saying "you should probably pause this now" — and importantly does
**not** flip the paused-by-solar flag, since it didn't actually do
anything. If you then pause it manually, that's your call.

### What if I manually resume / the cycle just ends?

If the paused-by-solar flag is on but the appliance is no longer paused
AND no longer running (i.e. it finished or you turned it off), the flag
clears itself. So the system never gets "stuck" thinking it's still
holding a pause.

---

## Pause modes

LG ThinQ exposes different controls per appliance:

| Appliance       | Mode             | Why                                          |
| --------------- | ---------------- | -------------------------------------------- |
| Washing machine | `select_option`  | `select.washer_operation` → `stop` to pause  |
| Tumble dryer    | `select_option`  | `select.dryer_operation` → `stop` to pause   |
| Dishwasher      | `notify_only`    | LG ThinQ exposes no control entities         |

> **LG ThinQ select options** for both washer and dryer are
> `start`, `stop`, `power_off`, `wake_up`. There is no `pause` option —
> `stop` is what pauses a running cycle, and `start` resumes it.
> Do **not** use `power_off` to pause: it ends the cycle.

> ⚠️ Do **not** use `switch_off` for washers or dishwashers — cutting
> power mid-cycle can leave water in the drum or trip the appliance.
> Even on the dryer, be aware that **most appliances do not auto-resume
> their program after a cold power cycle** — you may have to restart it
> manually. Prefer `select_option` whenever the appliance supports it.

---

## Helpers

### Required (per appliance)

- `input_boolean.<appliance>_paused_by_solar` — flag the blueprint sets
  when **it** paused the appliance (so it doesn't resume something you
  paused yourself).

### Required (shared, one set)

- `input_boolean.appliance_smart_control_enable` — master enable toggle.

### Optional but recommended (per appliance)

These improve robustness — especially across HA restarts. Each can be
left blank and the blueprint will fall back to less precise behaviour.

- `input_datetime.<appliance>_pause_timestamp` (date + time) —
  exact moment of last pause. More reliable than reading
  `last_changed` off the boolean (which resets on restart).
- `input_datetime.<appliance>_low_export_since` (date + time) —
  records when export first dropped below the threshold. Required for
  `pause_dwell` to actually do anything.
- `input_datetime.<appliance>_last_resume` (date + time) —
  remembers when an auto-resume last fired, so it can't fire twice
  on the same day.

### Optional (only for `fixed_time` resume)

- `input_datetime.<appliance>_solar_resume_time` (time only).

---

## Smarter gating sensors (all optional)

Provide whichever you have; the blueprint will use them to *avoid*
pausing when pausing wouldn't actually save anything.

| Input                              | What it does                                         |
| ---------------------------------- | ---------------------------------------------------- |
| `solar_forecast_remaining_sensor`  | kWh of solar still expected today (Forecast.Solar / Solcast). If it covers the rest of the cycle, skip pause. |
| `tomorrow_forecast_sensor`         | Expected kWh tomorrow. If below `min_tomorrow_kwh`, skip pause. |
| `tariff_sensor`                    | Current grid import price. If below `cheap_grid_threshold`, skip pause. |
| `battery_soc_sensor`               | Home battery SOC %. If at or above `battery_min_soc`, skip pause. *(Future-proof — set later when you add a battery.)* |
| `appliance_power_w`                | Nominal draw in watts. Used to estimate cycle kWh in notifications and forecast comparison. |

---

## Tunables you'll most likely touch

| Input                     | Default | Meaning                                      |
| ------------------------- | ------- | -------------------------------------------- |
| `lead_minutes`            | 60      | How early before sunset the pause window opens. |
| `pause_window_end_offset` | 5       | How long after sunset the window stays open. |
| `export_threshold`        | 300 W   | Below this = "we're importing significantly". |
| `pause_dwell`             | 5 min   | Anti-flap on cloud cover.                    |
| `min_remaining_minutes`   | 15      | Don't pause cycles that are nearly done.     |
| `max_remaining_minutes`   | 240     | Don't pause cycles that just started.        |
| `resume_dwell`            | 10 min  | Export must hold above threshold this long before auto-resume. |
| `earliest_resume_hour`    | 8       | No auto-resume before this hour.             |
| `min_pause_hours`         | 4       | No auto-resume sooner than this after pause. |

---

## Verifying your `select.*_operation` options

For reference, on a current LG ThinQ install the operation selects
expose:

- `select.washer_operation`: `start`, `stop`, `power_off`, `wake_up`
- `select.dryer_operation`:  `start`, `stop`, `power_off`, `wake_up`

If yours differ, open **Developer Tools → States**, look at the
`options` attribute on the select entity, and update the
`pause_option` / `resume_option` blueprint inputs accordingly.

---

## Installation

Copy `appliance-smart-control.yaml` into
`config/blueprints/automation/andre-sam/` (or import via the
My Home Assistant button once the repo is published) and create one
automation per appliance.

---

## Example: washer instance

- Status sensor: `sensor.washer_current_status`
- Remaining time: `sensor.washer_remaining_time`
- Pause mode: `select_option`
- Operation select: `select.washer_operation`
- Pause option: `stop`
- Resume option: `start`
- Paused flag: `input_boolean.washer_paused_by_solar`
- Pause timestamp: `input_datetime.washer_pause_timestamp`
- Low-export since: `input_datetime.washer_low_export_since`
- Last resume: `input_datetime.washer_last_resume`
- Resume strategy: `auto_export`
- Appliance power: `2000` W

## Example: dishwasher instance

- Status sensor: `sensor.dishwasher_current_status`
- Remaining time: `sensor.dishwasher_remaining_time`
- Pause mode: `notify_only`
- Notify service: `notify.mobile_app_<your_phone>`
- Resume strategy: `manual`
- Appliance power: `1800` W

The blueprint will notify you to suggest pausing it manually; once the
appliance leaves the running/paused state, no flag needs clearing
(notify-only never sets the flag in the first place).

---

## Troubleshooting

- **"Nothing pauses, ever."** Check the master enable boolean is on,
  the appliance status sensor is in your `running_states` list, and the
  enable conditions evaluate true in **Developer Tools → Template**.
- **"It pauses on a cloud."** Increase `pause_dwell` (and make sure
  `low_export_since` helper is configured — without it, dwell is 0).
- **"It resumed at 4 a.m."** Set `earliest_resume_hour` higher and
  configure `last_resume_date` so a second same-day resume can't fire.
- **"It paused something I'd already manually paused."** Shouldn't
  happen — the paused-by-solar flag gates this. If it does, you may
  have flipped the boolean by hand; turn it off and the next manual
  pause will be respected.
