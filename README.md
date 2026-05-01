# Appliance Smart Control

A Home Assistant **blueprint** that pauses long-running appliances
(dishwasher, washing machine, tumble dryer) shortly before sunset when
household solar export drops below a threshold, so the rest of the cycle
runs on free solar the next day instead of grid power.

Built around **LG ThinQ** entities but works with any appliance that
exposes a status sensor and either an operation `select` or a power
`switch`.

## How it works

Every minute the automation checks:

1. Is the appliance **running**?
2. Are we within `lead_minutes` (default 60) of **sunset**?
3. Is grid **export below `export_threshold`** (default 300 W)?
4. Is **more than `min_remaining_minutes`** (default 15) of cycle left?

If all four are true, it pauses the appliance and remembers it did so via
a per-appliance `input_boolean` flag. The next day it can auto-resume
when solar returns, at a fixed time, or wait for you to press resume.

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
> The `switch_off` mode is only safe for the dryer.

## Required helpers

Per appliance:

- `input_boolean.<appliance>_paused_by_solar` — flag the blueprint sets
  when **it** paused the appliance (so it doesn't resume something you
  paused yourself).

Shared (one set):

- `input_boolean.appliance_smart_control_enable` — master enable.
- *(optional, `fixed_time` resume)* `input_datetime.appliance_solar_resume_time`.

## Verifying your `select.*_operation` options

For reference, on a current LG ThinQ install the operation selects expose:

- `select.washer_operation`: `start`, `stop`, `power_off`, `wake_up`
- `select.dryer_operation`:  `start`, `stop`, `power_off`, `wake_up`

If yours differ, open **Developer Tools → States**, look at the
`options` attribute on the select entity, and update the
`pause_option` / `resume_option` blueprint inputs accordingly.

## Installation

Copy `appliance-smart-control.yaml` into
`config/blueprints/automation/andre-sam/` (or import via the
My Home Assistant button once the repo is published) and create one
automation per appliance.

## Example: washer instance

- Status sensor: `sensor.washer_current_status`
- Remaining time: `sensor.washer_remaining_time`
- Pause mode: `select_option`
- Operation select: `select.washer_operation`
- Pause option: `stop`
- Resume option: `start`
- Paused flag: `input_boolean.washer_paused_by_solar`
- Resume strategy: `auto_export`

## Example: dishwasher instance

- Status sensor: `sensor.dishwasher_current_status`
- Remaining time: `sensor.dishwasher_remaining_time`
- Pause mode: `notify_only`
- Notify service: `notify.mobile_app_<your_phone>`
- Resume strategy: `manual`

The blueprint will notify you to pause it manually; once you do, the
flag clears automatically when the appliance leaves the paused state.
