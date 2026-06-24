# Battery Sentinel (Alpha)

Two Home Assistant template blueprints that watch every battery device in the house and report, as sensors, which ones are running low and which ones have quietly stopped reporting.

Battery monitoring in Home Assistant has looked the same for years. The common blueprints are automation blueprints: they check a percentage on a schedule and push a message. The notification is the entire product. You get a one-time alert, and nothing else in your system can read the result, because there is no result to read, only an action that already fired.

That corner of the ecosystem has stalled. Little has moved in a long time. The forums carry a steady run of feature requests, list the actual battery type so the alert tells you what to buy, catch the sensor that died without going unavailable, put the low devices on a shopping list, send the alert somewhere other than one phone, and most of them go unanswered. They are not hard requests. They are simply out of reach for an automation whose only output is a notification, because a notification has nowhere to put a list, a history, or a second reader.

Battery Sentinel is a fresh approach, written from scratch. It is not a fork or a patch of anything. The starting point is different: instead of building down from the notification, it builds up from the sensor.

## Why a Sensor Instead of an Automation

These are template blueprints, so each one produces a `sensor`, not a one-time action. That single change is what unlocks everything the notification-only model cannot reach. A sensor has a state and attributes that persist, and anything in Home Assistant can read it. One sensor, many listeners:

- **A notification**, the obvious one. A companion automation blueprint for this is planned, so you get the polished alert without hand-writing the Jinja yourself.
- **A dashboard card** that lists the low devices with area and battery type, so a glance tells you what to grab from the drawer.
- **A to-do or shopping list**, each low cell dropped onto a list so the right battery is on hand before you make the swap.
- **A voice summary** through Assist, asking which batteries need attention before you head out.
- **A gate on another automation**, logic that only runs when the list is empty or non-empty, for example holding back a routine that leans on a sensor that has gone quiet.

The notification is the least of what the signal can drive. It is just the first reader people reach for.

## What It Does

Battery Sentinel splits the job across two single-purpose sensors so each one reports cleanly, with no attributes from the other muddying the output:

- **Battery Sentinel - Low** counts the batteries at or below a threshold.
- **Battery Sentinel - Not Reporting** counts the batteries that stopped reporting, including the one frozen at a healthy-looking value that a percentage check can never see.

Neither notifies on its own. Each is a signal: the state is a count, the flagged devices are listed in a `devices` attribute, and you decide what reads it. Build the pair for full coverage, or just the one you want.

**Delivering now:**

- Two single-purpose sensors, the state as a count, the offenders in a structured `devices` attribute carrying name, area, level or last-seen time, and battery type.
- Low handles both percentage and binary battery sensors, with a value-hysteresis band so a cell sitting at the threshold does not flap in and out of the list.
- Not Reporting judges liveness by the device, not the sticky battery reading, and judges it once a day so an honest daily reporter is not flagged for the hours it is quiet. The daily verdict is latched between runs.
- Scope by label, area, device, or entity, with include and exclude, and exclude always winning.
- Area and device scopes are filtered to real battery entities; labels and explicit entity lists are trusted as you curated them.
- Optional debug log line, and each sensor registered with a `unique_id` so you can rename it or place it in an area from the interface.

**Envisioned:**

- A companion notification automation blueprint, so the alert is a guided setup rather than hand-written templates.
- Voltage-reporting devices supported alongside percentage and binary.
- Actionable notifications, tap to drop the dead cell straight onto a to-do list.
- A combined health view that reads both sensors as a single signal.

These are alpha builds, under active testing on a live system. The design may still change. Do not rely on them for anything load-bearing until they graduate.

Full write-up and worked examples: <https://xeazy.com/battery-sentinel-blueprint/>  Questions and discussion: <https://xeazy.com/logbook/>

## Import the Blueprints

Each blueprint imports on its own. Use both for full coverage, or import only the one you want. They are separate entities and do not depend on each other.

### Battery Sentinel - Low

The low-battery counter. Import this one for the at-or-below-threshold list.

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Ftemplate%2Fbattery_sentinel_low.yaml)

Or paste this URL into Settings, Automations & Scenes, Blueprints, Import Blueprint:

```
https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/template/battery_sentinel_low.yaml
```

### Battery Sentinel - Not Reporting

The stopped-reporting counter. Import this one for the silent-failure list.

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Ftemplate%2Fbattery_sentinel_not_reporting.yaml)

Or paste this URL:

```
https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/template/battery_sentinel_not_reporting.yaml
```

Importing only registers a blueprint. It does not create a sensor yet. After import, the files land at `config/blueprints/template/TheThinkingHome/battery_sentinel_low.yaml` and `config/blueprints/template/TheThinkingHome/battery_sentinel_not_reporting.yaml`. Verify they are there, because you will need those paths in the next step.

## Setting Up the Sensors

The steps below are the same for both blueprints. Read them once, then jump to the section for whichever blueprint you are building.

### The Catch with Template Blueprints

Automation blueprints get a friendly Create automation button right on the Blueprints page. Template blueprints do not. You will not find these in the Create Helper list, and that is expected, not a fault. Template blueprints become real entities through a short piece of YAML called `use_blueprint`. It takes a path to the blueprint and a set of inputs:

```yaml
use_blueprint:
  path: TheThinkingHome/battery_sentinel_low.yaml
  input:
    low_threshold: 20
    sensor_name: Battery Sentinel - Low
    unique_id: battery_sentinel_low
```

`path` is relative to `config/blueprints/template/`, so it is the `TheThinkingHome` folder from the import step plus the filename. `unique_id` is the only required setting; the rest have sensible defaults. Every parameter is laid out in the reference table for each blueprint below.

That block has to sit under Home Assistant's `template:` configuration. The tidy way to do that, and the way that scales, is a package.

### What a Package Is, and How to Create One

A package is a single YAML file that holds a bundle of related configuration. Rather than scattering entities across your main files, you put everything for one feature into one file, and Home Assistant folds it into your overall configuration at startup. Packages are optional, but they keep related things together and easy to find.

If you have never used packages, switch them on once. In your `configuration.yaml`, add the two lines below. If you already have a `homeassistant:` section, just add the `packages:` line underneath it:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Then create a folder named `packages` next to your `configuration.yaml`. Every YAML file you drop in there is now a package.

Now make a package file, for example `packages/blueprint_battery_sentinel.yaml`, and put both `use_blueprint` blocks inside one `template:` list. Each needs its own `unique_id`, since reusing one collides the two sensors:

```yaml
template:
  - use_blueprint:
      path: TheThinkingHome/battery_sentinel_low.yaml
      input:
        low_threshold: 20
        low_clear_margin: 2
        scan_interval: "/2"
        sensor_name: Battery Sentinel - Low
        unique_id: battery_sentinel_low

  - use_blueprint:
      path: TheThinkingHome/battery_sentinel_not_reporting.yaml
      input:
        report_hour: 8
        lookback_hours: 28
        sensor_name: Battery Sentinel - Not Reporting
        unique_id: battery_sentinel_not_reporting
```

Note that the Not Reporting block has no `scan_interval`. It does not scan on an interval; it evaluates once a day at `report_hour` and on Home Assistant start. The Low block does take `scan_interval`. To narrow either sensor, add an `include_target` with a label, area, device, or entity, and an `exclude_target` for anything to drop. If you already keep template entities in your `configuration.yaml` under a `template:` key, you can drop the `use_blueprint` blocks into that list instead. The package is simply the cleaner home once you have a few.

### A Note on Scoping

Both sensors scope the same way, through `include_target` and `exclude_target`, and both accept entities, areas, devices, or labels. There is one behavioral difference worth knowing, because it decides what a scope actually watches.

An explicit entity list and a label are **trusted**. You hand-picked those, so the blueprint watches exactly what you named, without second-guessing it. That also means a label is only as clean as you keep it: tag a non-battery entity into the label and the blueprint will read it as though it were a battery.

An area or a device is **filtered to battery entities**. An area holds everything assigned to it, signal strength readings, connectivity sensors, blinds positions, and so on, and a device exposes many entities beyond its battery. Watching those raw would read a signal strength of -56 as if it were a battery at -56 percent. So when you scope by area or device, the blueprint keeps only the `device_class: battery` entities and drops the rest.

In short: name entities or a label and you get exactly that list; point at an area or device and you get the batteries within it. Exclude always wins over include, which is how you drop a phone, a tablet, or a device you are deliberately running down.

### Load It

A brand-new package file needs a full Home Assistant restart to register the first time. After that, adding more `use_blueprint` blocks to the same file only needs a quick reload through Developer Tools, YAML, Reload Template Entities.

When it comes up you will have a sensor named after each `sensor_name`. Watch them in Developer Tools, States. The state is the count, and the `devices` attribute lists the flagged devices.

# Battery Sentinel - Low

Counts the batteries at or below a threshold and lists them, with the percentage, area, and battery type for each.

## What You Get

A `sensor` whose state is the number of low batteries, with the offenders listed in a `devices` attribute.

- **Percentage and binary, both handled.** A device counts as low when its battery percentage is at or below the threshold, or when a binary battery sensor reads `on`. A binary low sensor has no number to report, so its `level` comes through as `null`.
- **Hysteresis so it does not flap.** A value-hysteresis band keeps a device hovering at the threshold from bouncing in and out of the list. It stays low until it climbs above the threshold plus a margin, which for a battery only happens when a fresh cell is fitted.
- **Structured output.** The `devices` list holds objects carrying the device name, area, percentage level, and battery type, so every consumer formats them its own way.

## What the Sensor Looks Like

With two cells below the threshold:

```yaml
sensor.battery_sentinel_low
State: 2

total_monitored: 27
last_evaluated: '2026-06-24T10:15:00-05:00'
devices:
  - name: Master Bath Motion
    entity_id: sensor.master_bath_motion_battery
    area: Master Bath
    level: 12
    battery_type: CR2450
  - name: Back Door
    entity_id: binary_sensor.back_door_battery
    area: Back Door
    level: null
    battery_type: ''
```

The state is the single number you badge or trigger on. The `devices` list is there for any consumer that wants the detail. `total_monitored` reports how many battery entities fell within the scope. A binary device shows `level: null`, since "on" carries no percentage.

## Parameters

| Input | Required | Default | Accepted values | What it does |
| --- | --- | --- | --- | --- |
| `unique_id` | Yes | none | any short text id, distinct per sensor | Registers the entity so you can rename it or place it in an area. Reusing one collides two sensors. |
| `low_threshold` | No | `20` | 1 to 100 | A battery percentage at or below this counts as low. Binary battery sensors reading `on` always count as low. |
| `low_clear_margin` | No | `2` | 0 to 25 | How far above the threshold a flagged device must climb before it clears. The hysteresis band that stops flapping. |
| `include_target` | No | empty | entities, areas, devices, or labels | Which devices to watch. Empty watches every battery device-class entity. Areas and devices are filtered to batteries; labels and entity lists are trusted. |
| `exclude_target` | No | empty | entities, areas, devices, or labels | Which to leave out. Exclude always wins over include. |
| `scan_interval` | No | `/2` | `/1`, `/2`, `/3`, `/5`, `/10`, `/15`, `/30` | How often the sensor re-scans. In the alpha this is in minutes for fast testing; it reverts to hours for production. |
| `sensor_name` | No | `Battery Sentinel - Low` | any text | The friendly name, and what the entity id is built from. |
| `debug_enabled` | No | `false` | on or off | When on, writes one diagnostic line to the system log each evaluation, naming the flagged devices. |

## A Note on Hysteresis

A battery hovering right at the threshold reports noise: 20.1, 19.9, 20.1. Without a guard, that flaps a device in and out of the low list. Battery Sentinel - Low puts a deadband in the value. A device enters the list when it drops below the threshold, and clears only when it climbs above the threshold plus the margin. Because a battery does not recover on its own, the only thing that pushes a flagged device past the clear point is a fresh cell, which is exactly when you want it to clear.

## Changelog: Battery Sentinel - Low

| Version | Notes |
| --- | --- |
| 1.0.0-alpha.2 | Area and device scopes are re-filtered to `device_class: battery`. Pointing the sensor at an area previously pulled in every entity there (signal strength, connectivity, blinds position) and read their numbers as battery levels. Explicit entity lists and labels stay trusted and unfiltered, since the user curates those. |
| 1.0.0-alpha | Initial build. Single-purpose low-battery sensor split from the combined Battery Sentinel blueprint, so the entity carries only low attributes. Value-hysteresis band so a device at the threshold does not flap. Include and exclude by entity, area, device, or label, with exclude winning. Percentage and binary battery sensors; voltage staged for later. Optional debug log line. Under active testing on a live system. |

# Battery Sentinel - Not Reporting

Counts the batteries that have stopped reporting, including the device frozen at a healthy-looking value, and lists each with how long it has been quiet.

## What You Get

A `sensor` whose state is the number of devices that have gone quiet, with the offenders listed in a `devices` attribute.

- **Judged by the device, not the battery reading.** A battery percentage is sticky: many devices only send a new value when the percentage actually changes, which on an efficient sensor can be once a week. Asking whether the battery entity reported recently would flag healthy devices constantly. Instead, the sensor takes the most recent report across all of the device's entities as the heartbeat. A live device always has something fresh, link quality, motion, temperature, even when its battery number has not moved in days. A genuinely dead device has everything stale.
- **Catches the silent failure.** "Not reporting" does not mean the literal `unavailable` state. It means nothing on the device has updated within the lookback window, whatever value it is still showing. A device frozen at eighty percent still reads `80`, and an `unavailable` check sails right past it. Asking whether the device reported at all catches both the explicitly-unavailable device and the silently-frozen one in a single test.
- **Judged once a day, then held.** The check runs once a day at an hour you choose, not continuously, so a water leak sensor that legitimately reports once a day is not flagged for the hours it is quiet. The verdict is latched in the sensor's own attributes and held until the next daily check, so it does not flap between evaluations.
- **Structured output.** The `devices` list carries the device name, area, last-seen time, a human-readable age, and battery type.

## What the Sensor Looks Like

With one device gone quiet:

```yaml
sensor.battery_sentinel_not_reporting
State: 1

total_monitored: 27
last_evaluated: '2026-06-24T08:00:00-05:00'
eval_date: '2026-06-24'
devices:
  - name: Guest Room Temperature
    entity_id: sensor.guest_room_temp_battery
    area: Guest Bedroom
    last_seen: '2026-06-21T03:14:08-05:00'
    age: 3 days
    battery_type: CR2032
```

The state is the number you badge or trigger on. `eval_date` records the day the last check ran; the sensor reads it back to hold the once-a-day verdict between scans. `last_seen` is the freshest report found across the whole device, and `age` is that gap in plain language.

## Parameters

| Input | Required | Default | Accepted values | What it does |
| --- | --- | --- | --- | --- |
| `unique_id` | Yes | none | any short text id, distinct per sensor | Registers the entity so you can rename it or place it in an area. Reusing one collides two sensors. |
| `report_hour` | No | `8` | 0 to 23 | The hour of day the check is judged. The sensor fires once a day at this hour, and on Home Assistant start. |
| `lookback_hours` | No | `28` | 1 to 168 | How many hours back from the check a device must have reported to count as alive. Raise it if normal daily reporters keep getting flagged. |
| `include_target` | No | empty | entities, areas, devices, or labels | Which devices to watch. Empty watches every battery device-class entity. Areas and devices are filtered to batteries; labels and entity lists are trusted. |
| `exclude_target` | No | empty | entities, areas, devices, or labels | Which to leave out. Exclude always wins over include. |
| `sensor_name` | No | `Battery Sentinel - Not Reporting` | any text | The friendly name, and what the entity id is built from. |
| `debug_enabled` | No | `false` | on or off | When on, writes one diagnostic line to the system log each evaluation, covering the daily-check timing and naming the flagged devices. |

## A Note on What "Not Reporting" Means

The phrase is doing real work, so it is worth pinning down. It is not the `unavailable` state, and it is not the battery percentage. It is whether the device behind the battery has sent any update, on any of its entities, within the lookback window.

That framing is what catches the failure people actually get stranded by. The device that froze at eighty percent still reads `80`, looks healthy on every dashboard, and is dead. A percentage check believes the `80`. An `unavailable` check believes the device is fine because Home Assistant never marked it unavailable. Asking instead "has anything on this device reported lately" sees through both, because a dead device goes silent across every entity it owns, not just its battery.

The once-a-day judging is the other half. A rolling twenty-four hour window would flag an honest once-a-day reporter every day it ran a little late. Judging once a day, against a lookback you can widen to twenty-eight hours or more, lets the daily reporter pass while a genuinely dead device is caught the next morning.

## Changelog: Battery Sentinel - Not Reporting

| Version | Notes |
| --- | --- |
| 1.0.0-alpha.4 | Area and device scopes are re-filtered to `device_class: battery`, matching the Low blueprint. An area sweeps in every entity it holds; without the filter, non-battery entities were treated as monitored. Explicit entity lists and labels stay trusted. |
| 1.0.0-alpha.3 | Removed the `scan_interval` input. The sensor now fires once a day at `report_hour` plus on Home Assistant start, instead of waking on a minute interval and doing nothing on most wakes. The daily latch still guards against redundant runs and lets a restart after the report hour catch up. |
| 1.0.0-alpha.2 | Judge liveness by the device, not the battery entity. A battery reading is sticky and may report only when the percentage changes, so the old per-entity check false-flagged healthy devices. Now the freshest report across all of a device's entities is the heartbeat, with a fallback to the battery entity when it has no device behind it. |
| 1.0.0-alpha | Initial build. Single-purpose stopped-reporting sensor split from the combined Battery Sentinel blueprint. Judged once a day against a lookback window, catching the device that freezes at its last value without going unavailable. The daily verdict is latched in the sensor's own attributes. Include and exclude by entity, area, device, or label, with exclude winning. Optional debug log line. Under active testing on a live system. |

# Putting the Signal to Work

Because each sensor is a count with a `devices` list, anything can read it. A few examples, all of which work the same whether you built one sensor or both.

A notification reading both sensors, speaking only when something needs attention:

```yaml
trigger:
  - trigger: state
    entity_id:
      - sensor.battery_sentinel_low
      - sensor.battery_sentinel_not_reporting
action:
  - action: notify.mobile_app_yourphone
    data:
      title: Battery attention needed
      message: >-
        {% set low = state_attr('sensor.battery_sentinel_low', 'devices') %}
        {% set nr = state_attr('sensor.battery_sentinel_not_reporting', 'devices') %}
        {% if low %}Low: {{ low | map(attribute='name') | join(', ') }}.{% endif %}
        {% if nr %}Not reporting: {{ nr | map(attribute='name') | join(', ') }}.{% endif %}
```

A dashboard card listing the low devices with area and battery type, so a glance tells you what to grab from the drawer:

```yaml
type: markdown
content: >-
  {% for d in state_attr('sensor.battery_sentinel_low', 'devices') %}
  - **{{ d.name }}** ({{ d.area }}) {{ d.level }}%, {{ d.battery_type }}
  {% endfor %}
```

A to-do consumer, dropping each low cell onto a shopping list so the right battery is on hand before the swap:

```yaml
trigger:
  - trigger: state
    entity_id: sensor.battery_sentinel_low
action:
  - repeat:
      for_each: "{{ state_attr('sensor.battery_sentinel_low', 'devices') }}"
      sequence:
        - action: todo.add_item
          target:
            entity_id: todo.shopping
          data:
            item: "{{ repeat.item.name }} battery ({{ repeat.item.battery_type }})"
```

The Not Reporting sensor earns its place with a case the others miss: a water leak detector reports once a day and then, one week, simply stops. Its last reading still shows a full battery, so a percentage check says all is well while the sensor that is supposed to catch a flood is dead. Battery Sentinel - Not Reporting flags it the next morning, and the leak sensor gets a fresh cell before it is ever needed.

For more worked examples, detailed setup, and the fuller story behind each design choice, see the article: <https://xeazy.com/battery-sentinel-blueprint/>

# License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>). These blueprints are free software: you may use, modify, and redistribute them under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). They are provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt them, keep this copyright and license notice intact.

You will find the full YAML for both blueprints, this README, and the one-click import badges at the [Thinking Home blueprints repository](https://github.com/TheThinkingHome/Automations).
