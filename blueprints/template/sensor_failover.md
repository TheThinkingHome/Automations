# Sensor Failover

A Home Assistant template sensor blueprint that wraps a primary numeric sensor with a pool of backups. When the primary is online you get its reading. When it goes offline you get the average of any backups still online, optionally weighted if some you trust more than others. When everything is offline you get a configurable default value, or the sensor reports unavailable.

A wireless temperature sensor reports 21°C most of the time, but its battery indicator is creeping down and it has started disappearing for an hour at a time. Another sensor across the same room reads 21.3°C when the first one is up. The two are close enough that when one is missing the other can stand in. Or take an outdoor lux sensor mounted at the eave that drops out for ten minutes every cold morning. The reading you actually want, "how bright is it outside," is more reliable than any single sensor can be on its own; pair it with two other lux readings and the value never disappears.

This blueprint creates that wrapper sensor. You designate a primary, the trusted source. You list one or more backups, sensors that can stand in. You can optionally weight the backups if some are more trustworthy than others. You can optionally set a default value for when everything is offline, or let the sensor report unavailable in that case. The resulting sensor is a regular Home Assistant numeric sensor that downstream automations and dashboards treat exactly like any other reading. The unreliability of any single source is invisible to them.

Full write-up and the longer story behind the design: <https://xeazy.com/sensor-failover-blueprint/>  Questions and discussion: <https://xeazy.com/logbook/d/39-sensor-failover-blueprint>

## What You Get

- The primary's reading is used whenever it is available.
- A pool of backups gets averaged when the primary is offline. Backups that are themselves offline are skipped; the average is over the ones still online.
- Optional weights for the backups if some are more trustworthy than others. Comma-separated numbers, one per backup, in the same order. Blank means equal weighting.
- An optional default value for the case where the primary and all backups are offline at once. Blank means the sensor reports unavailable instead.
- A `device_class` so the resulting sensor displays correctly and groups with similar sensors. Defaults to `illuminance` (matching the original lux use case); override with `temperature`, `humidity`, `power`, `energy`, or any other valid sensor device class.
- A `state_class` so Home Assistant's statistics engine knows whether the reading is a `measurement`, a `total_increasing` counter, or a resettable `total`. Defaults to `measurement`, which is right for most numeric sensors.
- An `active_source` attribute that reports `primary`, `backups`, `default`, or `none`, so a dashboard tile or anyone debugging can see at a glance which source is feeding the value.
- An `online_backups` attribute that lists the backup entities currently contributing. Empty when none are available.
- A unique_id so the sensor is registered: rename it in the interface, place it in an area, all from the UI.

A sensor is treated as "offline" when its state is `unavailable`, `unknown`, blank, or not a number. The blueprint never feeds a malformed value into the output.

## Where This Fits

The closest built-in alternative is the Min/Max helper, which takes multiple sensors and outputs their min, max, mean, median, or sum. It is the right tool when every sensor is a peer and you want one number out of the group. Where it falls short is the failover model: it averages every sensor every time, so a flaky sensor that bounces between 200 lx and `unavailable` pulls the average around when it should just be ignored. There is no primary, no hierarchy of trust, no last-resort default, and no way to see which source actually contributed.

A hand-written template sensor can do anything this blueprint does, but you write the template and handle the edge cases yourself. The unavailable-handling, the weighted-average math, the renormalization when some backups drop out, the default fallback, the attribute reporting; that is a hundred lines of Jinja you write once and probably should not write three times.

This blueprint sits where you want a numeric sensor with a clear hierarchy of trust (one primary, a pool of backups, an optional default), and you want the unreliability of any single source to be invisible to the automations and dashboards that consume it.

## Requirements

Home Assistant 2026.5.4 or newer. That is the version it is verified on.

## Step 1: Import the Blueprint

The quick way, one click:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTheThinkingHome%2FAutomations%2Fblob%2Fmain%2Fblueprints%2Ftemplate%2Fsensor_failover.yaml)

Or by hand:

1. Go to Settings, then Automations & Scenes, then the Blueprints tab.
2. Click Import Blueprint at the bottom right.
3. Paste this URL and click Preview:

```
https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/sensor_failover.yaml
```

4. Confirm and Import.

After import, the blueprint appears on the Blueprints page.

## Step 2: The Catch with Template Blueprints

Automation blueprints get a friendly Create button right on the Blueprints page. Template blueprints do not. You will not find Sensor Failover anywhere in the Create Helper list, and that is expected, not a fault. Template blueprints become real entities through a short piece of YAML called `use_blueprint`. It takes a path to the blueprint and a set of inputs:

```yaml
use_blueprint:
  path: TheThinkingHome/sensor_failover.yaml
  input:
    primary_entity: sensor.fp2_living_room_illuminance
    backup_entities:
      - sensor.zigbee_lux_north
      - sensor.zigbee_lux_south
    sensor_name: Living Room Lux Failover
    unique_id: living_room_lux_failover
    unit_of_measurement: lx
    device_class: illuminance
```

`path` is relative to `config/blueprints/template/`, so it is the `<author>` folder from Step 1 plus the filename. `primary_entity`, `backup_entities`, `unique_id`, and `unit_of_measurement` are the required settings; `device_class` defaults to `illuminance` and must be corrected for any other sensor type (`temperature`, `humidity`, `power`, and so on). The other inputs have sensible defaults. For weighted averaging across backups and a configurable default-when-everything-is-offline value, see the optional inputs in the reference below.

That block has to sit under Home Assistant's `template:` configuration. The tidy way to do that, and the way that scales once you make more of these, is a package.

## Step 3: What a Package Is, and How to Create One

A package is a single YAML file that holds a bundle of related configuration. Rather than scattering a sensor here and an automation there across your main files, you put everything for one feature into one file, and Home Assistant folds it into your overall configuration at startup. Packages are optional, but they keep related things together and easy to find.

If you have never used packages, switch them on once. In your `configuration.yaml`, add the two lines below. If you already have a `homeassistant:` section, just add the `packages:` line underneath it:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Then create a folder named `packages` next to your `configuration.yaml`. Every YAML file you drop in there is now a package.

Now make a package file, for example `packages/sensor_failover.yaml`, and put your `use_blueprint` block inside a `template:` list:

```yaml
template:
  - use_blueprint:
      path: TheThinkingHome/sensor_failover.yaml
      input:
        primary_entity: sensor.fp2_living_room_illuminance
        backup_entities:
          - sensor.zigbee_lux_north
          - sensor.zigbee_lux_south
        sensor_name: Living Room Lux Failover
        unique_id: living_room_lux_failover
        unit_of_measurement: lx
        device_class: illuminance
        state_class: measurement
        backup_weights: ""
        default_value: ""
```

Each sensor you build from the blueprint is one more list item in the same file. They can wrap different primaries, with different backup pools, different weights, and different defaults, all in one place. Give each one its own `unique_id`, since reusing one collides the two sensors.

If you already keep template entities in your `configuration.yaml` under a `template:` key, you can drop the `use_blueprint` block into that list instead. The package is simply the cleaner home once you have a few.

## Step 4: Load It

A brand-new package file needs a full Home Assistant restart to register the first time. After that, adding more `use_blueprint` blocks to the same file only needs a quick reload through Developer Tools, YAML, Reload Template Entities. If you put the block in `configuration.yaml` directly under an existing `template:` key, you can skip the restart and just reload template entities.

When it comes up you will have a `sensor` named after your `sensor_name`. Its entity id is derived from the slugified name: `Living Room Lux Failover` becomes `sensor.living_room_lux_failover`. Watch it in Developer Tools, States: the value follows the primary's reading when the primary is online, and flips to the backup pool's average (or to your default value, if set) when the primary is unavailable. The `active_source` attribute reports `primary`, `backups`, `default`, or `none`, so you can see at a glance which tier is feeding the value.

## A Small Note on Why Primary Wins

There are a few design choices worth understanding. The first is that the primary takes priority whenever it is available, even if a backup is also online. The blueprint does not blend them.

The reason is trust. You designate one source as the primary because it is the one you actually want to read. The backups exist for the case where that source is unavailable. Blending the primary with backups during normal operation would mean every reading from your trusted sensor gets averaged with sensors that, by construction, are not as good. That is worse, not better.

If you find yourself wanting blending during normal use, the right answer is probably a Min/Max helper, not this blueprint. Min/Max treats every member as equal, which is correct when there is no reason to prefer one over another.

## A Small Note on Calibration

You may notice that two sensors measuring the same physical quantity often report different values. An outdoor lux sensor in direct sun reads 100,000 lx while an indoor lux sensor a meter inside the window reads 200 lx. They are both correct for their position, but they are not interchangeable. When the outdoor sensor drops and the indoor one stands in, the reading suddenly jumps to a fraction of what it was.

This blueprint deliberately does not try to calibrate the backups to match the primary. Calibration requires historical state, knowing the ratio between the two readings over time, and a template sensor cannot carry historical state. Even if it could, the ratio is wildly variable: it depends on blinds, sun direction, season, time of day. A learned-once offset would be wrong half the time.

If your backups read in a meaningfully different range from your primary, the practical workarounds are to pick backups that read in the same ballpark (two indoor sensors in the same room rather than one indoor and one outdoor), or to accept that the fallback reading is approximate. The blueprint's job is to keep the value flowing through outages, not to invent precision that the backup sensors do not have.

## A Small Note on What Counts as Offline

The blueprint treats a sensor as offline in four cases: the state is `unavailable`, the state is `unknown`, the state is blank, or the state is not a number. The first two are Home Assistant's standard markers for "I cannot reach this device." The third covers the case where an integration is set up but has never reported a value. The fourth covers the unusual case where a numeric sensor reports a string like `low_battery` or `calibrating`.

In all four cases the blueprint moves on to the next source as if the entity did not exist for this evaluation. Nothing is ever fed forward as 0 or any other sentinel value. If you see your wrapper sensor reading 0.0 unexpectedly, it is because one of your real sources is actually reading 0.0, not because of a malformed-state coverup.

## Worked Example

The original use case: an Aqara FP2 presence sensor reports a lux value that is reliable when the sensor is up but drops to `unavailable` several times a day. Two other lux sensors in the same room are less precise but are always online.

```yaml
template:
  - use_blueprint:
      path: TheThinkingHome/sensor_failover.yaml
      input:
        primary_entity: sensor.fp2_living_room_illuminance
        backup_entities:
          - sensor.zigbee_lux_north
          - sensor.zigbee_lux_south
        sensor_name: Living Room Lux Failover
        unique_id: living_room_lux_failover
        unit_of_measurement: lx
        device_class: illuminance
        state_class: measurement
        backup_weights: "70, 30"
        default_value: ""
```

This says: trust the FP2 when available, fall back to a weighted average of the two Zigbee lux sensors when it is not (70% weight to the north sensor, 30% to the south), report unavailable if everything is offline at once, and register the sensor with Home Assistant as an illuminance measurement so it displays correctly and feeds into the statistics engine.

Downstream automations that previously read `sensor.fp2_living_room_illuminance` directly can be repointed at `sensor.living_room_lux_failover` and stop encountering `unavailable`. A dashboard tile can show the new sensor with an attribute display for `active_source` so you can see when the FP2 has dropped out.

For more worked examples, including temperature and humidity scenarios and the longer story behind each design choice, see the article: <https://xeazy.com/sensor-failover-blueprint/>

## Parameters

| Input | Required | Default | Accepted values | What it does |
| --- | --- | --- | --- | --- |
| `primary_entity` | Yes | none | any `sensor` | The trusted source. Used whenever it is available. |
| `backup_entities` | Yes | none | one or more `sensor` entities | Sources used when the primary is offline. Averaged when more than one is online. |
| `sensor_name` | No | `Sensor Failover` | any text | The friendly name shown on the resulting sensor. Set this distinctly for each sensor you build, otherwise multiple sensors share the default friendly name in the UI even though their entity ids stay separate. |
| `unique_id` | Yes | none | any text, must be unique across your config | A unique id so the sensor is registered and can be renamed or placed in an area from the interface. Use a distinct value for every sensor you build, for example `living_room_lux_failover`. |
| `unit_of_measurement` | Yes | none | any text | The unit shown on the resulting sensor, for example `lx`, `°C`, or `%`. |
| `device_class` | Yes | `illuminance` | any valid Home Assistant sensor device class | Sets the device class on the resulting sensor, which affects display, grouping, and unit validation. Defaults to `illuminance` for the original lux use case; override for any other sensor type. Common values: `illuminance`, `temperature`, `humidity`, `pressure`, `power`, `energy`, `battery`, `current`, `voltage`, `co2`, `pm25`, `moisture`, `signal_strength`. See the [Home Assistant sensor device classes](https://www.home-assistant.io/integrations/sensor/#device-class) for the full list. If your numeric sensor measures something obscure, choose the closest match. |
| `state_class` | No | `measurement` | `measurement`, `total_increasing`, `total`, `measurement_angle` | Sets the state class for statistics. `measurement` is correct for most readings that go up and down (lux, temperature, humidity). `total_increasing` is for cumulative counters that only grow (energy meters). `total` is for resettable counters. |
| `backup_weights` | No | blank | comma-separated numbers, one per backup, in the same order | Weighted average when filled, equal weighting when blank. If the count of weights does not match the count of backups, any weight is not a number, or every weight is zero, the blueprint falls back to equal weighting. |
| `default_value` | No | blank | any number, or blank | The value returned when the primary and all backups are offline. Blank means the sensor reports unavailable in that case. |

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>)

This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.
