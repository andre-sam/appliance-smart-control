# Appliance Smart Control

A Home Assistant **blueprint** that pauses long-running appliances
(washing machine, tumble dryer, dishwasher) whenever live solar
production drops below a threshold, and resumes them once production
returns. One simple rule handles both "started too early in the
morning" and "still running into the evening".

Built around **LG ThinQ** entities but works with anything exposing a
status sensor and either an operation `select` or a power `switch`.

---

## How it works

A live solar production sensor (e.g. `sensor.power_production_now`) is
the single source of truth.

- If the appliance is **running** and production has been **below
  `production_threshold`** (default 1000 W) for `dwell_minutes`
  (default 5), pause it.
- If the appliance is **paused-by-us** and production has been **above
  the threshold** for `dwell_minutes`, resume it.
- If the user manually resumes or the cycle ends, the paused-by-solar
  flag clears itself.

That's it. No sunset windows, no forecasts, no time-of-day rules.
Production reading drives everything.

### Anti-flap

Pause and resume both use Home Assistant's `numeric_state` trigger with
a `for:` clause, so a passing cloud (or a brief sun spike at dawn) won't
cause toggling.

### Optional "almost finished" guard

If you provide `remaining_time_sensor`, the pause is skipped when fewer
than `min_remaining_minutes` are left — no point pausing something
that's nearly done.

### Optional Remote Start gate

If you provide `remote_start_sensor` (e.g.
`binary_sensor.dryer_remote_start`), the blueprint won't try to send
pause/resume commands while Remote Start is OFF — the LG API would
reject them anyway.

---

## Pause modes

| Appliance       | Mode             | Why                                         |
| --------------- | ---------------- | ------------------------------------------- |
| Washing machine | `select_option`  | `select.washer_operation` → `stop` to pause |
| Tumble dryer    | `select_option`  | `select.dryer_operation` → `stop` to pause  |
| Dishwasher      | `notify_only`    | LG ThinQ exposes no control entities        |

> LG ThinQ washer/dryer operation options are `start`, `stop`,
> `power_off`, `wake_up`. Use **`stop`** to pause and **`start`** to
> resume. Never use `power_off` to pause — that ends the cycle.

> ⚠️ `switch_off` is dryer-only at best, and most appliances do **not**
> auto-resume after a power cycle. Prefer `select_option` whenever
> available.

> ⚠️ **LG dryer sleep mode**: about 10 min after pausing, the dryer's
> status changes to `sleep` and the API rejects further commands. This
> simple blueprint won't recover it. If a pause is likely to last more
> than ~10 min, expect that the dryer needs a manual tap on the panel
> to resume. (A keepalive workaround is possible but not in this
> simplified version — re-introduce later if needed.)

---

## Required helpers

### Per appliance

- `input_boolean.<appliance>_paused_by_solar` — flag the blueprint sets
  when it pauses the appliance, so it never resumes something the user
  paused manually.

### Shared (one-off)

- `input_boolean.appliance_smart_control_enable` — master enable.

### YAML form

```yaml
input_boolean:
  appliance_smart_control_enable:
    name: Appliance Smart Control enabled
    icon: mdi:solar-power
  washer_paused_by_solar:
    name: Washer paused by solar
    icon: mdi:washing-machine
  dryer_paused_by_solar:
    name: Dryer paused by solar
    icon: mdi:tumble-dryer
  dishwasher_paused_by_solar:
    name: Dishwasher paused by solar
    icon: mdi:dishwasher
```

After editing: **Developer Tools → YAML → Reload Input Booleans**.

---

## Tunables

| Input                    | Default | Meaning                                  |
| ------------------------ | ------- | ---------------------------------------- |
| `production_threshold`   | 1000 W  | Pause below, resume above.               |
| `dwell_minutes`          | 5       | Anti-flap: must be sustained this long.  |
| `min_remaining_minutes`  | 10      | Don't pause cycles nearly done (only used if `remaining_time_sensor` is set). |

---

## Installation

Copy `appliance-smart-control.yaml` into
`config/blueprints/automation/andre-sam/` (or import via the
My Home Assistant button once the repo is published) and create one
automation per appliance.

---

## Example: washer instance

- Master enable: `input_boolean.appliance_smart_control_enable`
- Status sensor: `sensor.washer_current_status`
- Remaining time: `sensor.washer_remaining_time` *(optional)*
- Production sensor: `sensor.power_production_now`
- Production threshold: `1000`
- Dwell minutes: `5`
- Pause mode: `select_option`
- Operation select: `select.washer_operation`
- Pause option: `stop`
- Resume option: `start`
- Paused flag: `input_boolean.washer_paused_by_solar`
- Notify: `notify.mobile_app_<your_phone>`
- Friendly name: `Washer`

## Example: dryer instance

- As washer, but:
  - Status sensor: `sensor.dryer_current_status`
  - `paused_states`: `pause,sleep` *(treat sleep as still paused)*
  - Operation select: `select.dryer_operation`
  - Paused flag: `input_boolean.dryer_paused_by_solar`
  - Remote-start sensor: `binary_sensor.dryer_remote_start`

## Example: dishwasher instance

- Master enable: `input_boolean.appliance_smart_control_enable`
- Status sensor: `sensor.dishwasher_current_status`
- Production sensor: `sensor.power_production_now`
- Pause mode: `notify_only`
- Notify: `notify.mobile_app_<your_phone>`
- Friendly name: `Dishwasher`
- Paused flag: `input_boolean.dishwasher_paused_by_solar`

You'll get a notification suggesting you pause it manually.

---

## Troubleshooting

- **Nothing happens.** Check the master enable boolean is on, and the
  status sensor's current state is in your `running_states` list.
  Verify in Developer Tools → Template by pasting
  `{{ states('sensor.washer_current_status') }}`.
- **It's flapping.** Increase `dwell_minutes`.
- **It paused near the end of a cycle.** Configure
  `remaining_time_sensor` and `min_remaining_minutes`.
- **Resume command was rejected.** If you have an LG washer/dryer with
  Remote Start, configure `remote_start_sensor` so the blueprint won't
  try to send commands while Remote Start is disarmed.
- **Dryer didn't recover from `sleep`.** Known LG limitation in the
  simplified version. Either resume manually, or re-introduce keepalive
  logic (not included here).
