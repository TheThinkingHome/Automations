# Battery Sentinel (Alpha)

A Home Assistant template blueprint that watches every battery device in the house and reports, as one sensor, which ones are running low and which ones have quietly stopped reporting.

Most battery blueprints check a number once a day and send a message. That misses the failure that actually strands you. A battery does not always fade gracefully from twenty percent down to five. Sometimes it sits reading eighty percent for days because eighty was the last value it ever sent, while the device is dead and Home Assistant never marked it unavailable because nothing told it to. A percentage check cannot see that, and a plain unavailable check cannot either. Battery Sentinel watches for both failures: the cell that dropped below a threshold, and the device that went silent while still showing a healthy number.

It does not notify on its own. It is a signal. The sensor reports the state of the fleet as a count, with the problem devices listed in its attributes, and you decide what reads that signal: a notification, a dashboard card, a to-do entry, or any automation you write. A companion automation blueprint, Battery Sentinel Companion, is coming for people who want a turnkey notifier, but the sensor stands on its own.

This is the 1.0.0-alpha build, under active testing on a live system. The design may still change. Do not rely on it for anything load-bearing until it graduates.

Full write-up and worked examples: https://xeazy.com/battery-sentinel-blueprint/  Questions and discussion: https://xeazy.com/logbook/

## What You Get

A `sensor` whose state is a count of battery devices that need attention, with the offenders listed in structured attributes.

- **Three monitor modes.** Count low devices only, silent devices only, or both. Because this is a template blueprint, build it more than once, one sensor per purpose, for example a low sensor for a dashboard badge and a separate silent sensor that drives its own alert.
- **Low detection with hysteresis.** A battery at or below the threshold, or a binary battery sensor reading `on`, counts as low. A value-hysteresis band keeps a device hovering at the threshold from flapping in and out of the list. A flagged device stays low until it climbs above the threshold plus a margin, which for a battery only happens when a fresh cell is fitted.
- **Silent detection that respects daily reporters.** A device that has not reported within a lookback window counts as silent, judged once a day at an hour you choose rather than continuously. A water leak sensor that legitimately reports once a day is not flagged for the hours it is quiet, while a genuinely dead device is caught.
- **Structured attributes, not a pre-joined string.** The `low` and `silent` lists are objects, each carrying the device name, area, level or last-seen time, and battery type. Every consumer formats them its own way. This is the specific thing that broke older battery blueprints: they built the message text inside the template, and the empty-list and truncation bugs followed. Here the sensor reports data and the consumer writes the words.
- **Include and exclude by label, area, device, or entity.** Leave include empty to watch every battery device-class entity automatically. Set a label to watch only the devices you tag, which is the clean answer to a house with thousands of auto-added entities. Exclude always wins, for phones, tablets, or a device you are deliberately running down.
- **Percentage and binary battery sensors** are supported now. Voltage-reporting devices are staged for a later build.

## Requirements

Home Assistant 2026.5.4 or newer. That is the version it is verified on. No custom integrations are required. If you run the Battery Notes integration, battery types are picked up where exposed; if you do not, the battery type field is simply blank.

## Step 1: Import the Blueprint

The quick way, one click:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Ftemplate%2Fbattery_sentinel.yaml)

Or by hand:

1. Go to Settings, then Automations & Scenes, then the Blueprints tab.
2. Click Import Blueprint at the bottom right.
3. Paste this URL and click Preview:
   `https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/template/battery_sentinel.yaml`
4. Confirm and Import.

Importing only registers the blueprint. It does not create a sensor yet. After import, the file lands at `config/blueprints/template/<author>/<blueprint_package>`. This blueprint should be located at `TheThinkingHome/battery_sentinel.yaml`. Before continuing, verify that the Battery Sentinel blueprint is installed at this location, because you will need that path in the next step.

## Step 2: The Catch with Template Blueprints

Automation blueprints get a friendly Create automation button right on the Blueprints page. Template blueprints do not. You will not find Battery Sentinel anywhere in the Create Helper list, and that is expected, not a fault. Template blueprints become real entities through a short piece of YAML called `use_blueprint`. It takes a path to the blueprint and a set of inputs:

```yaml
use_blueprint:
  path: TheThinkingHome/battery_sentinel.yaml
  input:
    monitor_mode: both
    low_threshold: 20
    silent_report_hour: 8
    sensor_name: Battery Sentinel
    unique_id: battery_sentinel_all
```

`path` is relative to `config/blueprints/template/`, so it is the `<author>` folder from Step 1 plus the filename. `unique_id` is the only required setting; the rest have sensible defaults. Every parameter is laid out in the reference below.

That block has to sit under Home Assistant's `template:` configuration. The tidy way to do that, and the way that scales once you make more than one, is a package.

## Step 3: What a Package Is, and How to Create One

A package is a single YAML file that holds a bundle of related configuration. Rather than scattering a sensor here and an automation there across your main files, you put everything for one feature into one file, and Home Assistant folds it into your overall configuration at startup. Packages are optional, but they keep related things together and easy to find.

If you have never used packages, switch them on once. In your `configuration.yaml`, add the two lines below. If you already have a `homeassistant:` section, just add the `packages:` line underneath it:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Then create a folder named `packages` next to your `configuration.yaml`. Every YAML file you drop in there is now a package.

Now make a package file, for example `packages/battery_sentinel.yaml`, and put your `use_blueprint` block inside a `template:` list:

```yaml
template:
  - use_blueprint:
      path: TheThinkingHome/battery_sentinel.yaml
      input:
        monitor_mode: both
        low_threshold: 20
        low_clear_margin: 2
        silent_report_hour: 8
        silent_lookback_hours: 24
        scan_interval: "/2"
        sensor_name: Battery Sentinel
        unique_id: battery_sentinel_all
```

Each sensor you build from the blueprint is one more list item in the same file. Because the multi-instance pattern is the whole point of a sensor-first design, you will often want more than one. Give each one its own `unique_id`, since reusing one collides the two sensors. Here is a low sensor and a silent sensor side by side, the low one watching only labeled infrastructure and the silent one watching everything:

```yaml
template:
  # Low only, scoped to devices labeled "battery-monitored", phone excluded
  - use_blueprint:
      path: TheThinkingHome/battery_sentinel.yaml
      input:
        monitor_mode: low
        low_threshold: 20
        low_clear_margin: 2
        scan_interval: "/2"
        sensor_name: Battery Low
        unique_id: battery_sentinel_low
        include_target:
          label_id:
            - battery_monitored
        exclude_target:
          entity_id:
            - sensor.pixel_phone_battery

  # Silent only, watching every battery device-class entity, checked at 8 AM
  - use_blueprint:
      path: TheThinkingHome/battery_sentinel.yaml
      input:
        monitor_mode: silent
        silent_report_hour: 8
        silent_lookback_hours: 28
        scan_interval: "/2"
        sensor_name: Battery Silent
        unique_id: battery_sentinel_silent
```

If you already keep template entities in your `configuration.yaml` under a `template:` key, you can drop the `use_blueprint` block into that list instead. The package is simply the cleaner home once you have a few.

## Step 4: Load It

A brand-new package file needs a full Home Assistant restart to register the first time. After that, adding more `use_blueprint` blocks to the same file only needs a quick reload through Developer Tools, YAML, Reload Template Entities.

When it comes up you will have a sensor named after your `sensor_name`. Watch it in Developer Tools, States. The state is the count of devices needing attention, and the `low` and `silent` attributes list them. Drop a battery below the threshold and it appears in `low`; let a device go quiet past the lookback window and, at the next daily check hour, it appears in `silent`.

## What the Sensor Looks Like

On a day with two low cells and one device gone quiet, in both mode:

```yaml
sensor.battery_sentinel
State: 3

monitor_mode: both
total_monitored: 27
last_evaluated: '2026-06-24T10:15:00-05:00'
low_count: 2
silent_count: 1
low:
  - name: Master Bath Motion
    entity_id: sensor.master_bath_motion_battery
    area: Master Bath
    level: 12
    battery_type: CR2450
  - name: Front Gate Contact
    entity_id: sensor.front_gate_contact_battery
    area: Entry
    level: 18
    battery_type: AAA
silent:
  - name: Guest Room Temperature
    entity_id: sensor.guest_room_temp_battery
    area: Guest Bedroom
    last_seen: '2026-06-21T03:14:08-05:00'
    age: 3 days
    battery_type: CR2032
silent_eval_date: '2026-06-24'
```

The state is the single number you badge or trigger on. The lists are there for any consumer that wants the detail.

## Where This Fits

The most-used battery blueprint in the ecosystem checks a percentage on a schedule and notifies. It works, and for a small house it is enough. Battery Sentinel sits a step past it in two ways.

First, it finds the silent failure, not just the low one. The device frozen at eighty percent that quietly died is the one that bites, because the number looks fine right up until you need the sensor and it is not there. A percentage check never sees it. Scoping a stale-timestamp check to battery devices catches it, and the once-a-day judging keeps honest daily reporters from crying wolf.

Second, it is a sensor, not a notifier. A notification blueprint locks you into one message format and one delivery path. A sensor is read by anything: a notification, yes, but also a dashboard, a to-do list, a voice summary, a REST export, or an automation that only acts when the silent list is non-empty. The notification is the least of what the signal can drive. If you do want a ready-made notifier, that is what the coming Battery Sentinel Companion is for.

## Parameters

| Input | Required | Default | Accepted values | What it does |
| --- | --- | --- | --- | --- |
| `unique_id` | Yes | none | any short text id, distinct per sensor | Registers the entity so you can rename it or place it in an area from the interface. Reusing one collides two sensors. |
| `monitor_mode` | No | `both` | `low`, `silent`, `both` | What the state counts: low devices only, silent devices only, or either. |
| `low_threshold` | No | `20` | 1 to 100 | A battery percentage at or below this counts as low. Binary battery sensors reading `on` always count as low. |
| `low_clear_margin` | No | `2` | 0 to 25 | How far above the threshold a flagged device must climb before it clears. The hysteresis band that stops flapping. |
| `silent_report_hour` | No | `8` | 0 to 23 | The hour of day the silent check is judged. Silence is evaluated once per day at this hour, not continuously. |
| `silent_lookback_hours` | No | `24` | 1 to 168 | How many hours back from the check hour a device must have reported to count as alive. Raise it if normal daily reporters keep getting flagged. |
| `include_target` | No | empty | entities, areas, devices, or labels | Which devices to watch. Empty watches every battery device-class entity automatically. A label here watches only tagged devices. |
| `exclude_target` | No | empty | entities, areas, devices, or labels | Which devices to leave out. Exclude always wins over include. |
| `scan_interval` | No | `/2` | `/1`, `/2`, `/3`, `/4`, `/6`, `/12` | How often the sensor re-scans, in hours. Every 2 hours is light and plenty for batteries. |
| `sensor_name` | No | `Battery Sentinel` | any text | The friendly name, and what the entity id is built from. |

## A Small Note on Silent Detection

Silent does not mean the literal `unavailable` state. It means the device has not sent an update within the lookback window, whatever value it is still showing. That is deliberate, and it is the point of the feature. A device that froze at eighty percent still reads `80`, and an `unavailable` check sails right past it. Asking instead whether the device reported at all catches both the explicitly-unavailable device and the silently-frozen one in a single test.

The check is judged once a day at your chosen hour, against the lookback window, rather than on a rolling basis. A rolling twenty-four hour window would flag a once-a-day reporter every day it ran a little late. Judging once a day, with a lookback you can widen to twenty-eight hours or more, lets an honest daily reporter pass while a genuinely dead device is caught. The verdict is held until the next daily check, so it does not flap between scans.

## A Small Note on Hysteresis

A battery hovering right at the threshold reports noise: 20.1, 19.9, 20.1. Without a guard, that flaps a device in and out of the low list. Battery Sentinel puts a deadband in the value. A device enters the low list when it drops below the threshold, and clears only when it climbs above the threshold plus the margin. Because a battery does not recover on its own, the only thing that pushes a flagged device past the clear point is a fresh cell, which is exactly when you want it to clear. The sensor reads its own previous low list to make this decision, so a device that tripped at 19.9 stays low through a 20.1 bounce instead of chattering.

## A Small Note on the Alpha

These are the things still in flight, listed honestly because this is an alpha.

- **Voltage sensors are not yet handled.** Devices that report battery as a voltage rather than a percentage are skipped for now. Percentage and binary come first; voltage is the next layer, with per-voltage thresholds.
- **Attributes are uniform across modes.** A low-mode sensor still carries `silent_count` and `silent`, set to zero and empty. Per-mode key omission fights Home Assistant's static attribute schema, so for now the keys are constant and the `monitor_mode` attribute tells you what the sensor is actually watching.
- **The re-evaluation interval is hour-based.** Sub-hour cadences are not in this build. Hourly is the floor.
- **The silent list is frozen between daily checks.** Because silence is judged once a day, the `silent` list and its age strings hold the last check's values until the next one. This is intended, but the age figures are a snapshot, not live.
- **Battery type is best-effort.** It is read from a `battery_type` attribute where present. Deeper Battery Notes support is a later enhancement.

## Example Use

A notification consumer, reading a both-mode sensor and only speaking when something needs attention:

```yaml
trigger:
  - trigger: numeric_state
    entity_id: sensor.battery_sentinel
    above: 0
action:
  - action: notify.mobile_app_yourphone
    data:
      title: Battery attention needed
      message: >-
        {% set low = state_attr('sensor.battery_sentinel', 'low') %}
        {% set silent = state_attr('sensor.battery_sentinel', 'silent') %}
        {% if low %}Low: {{ low | map(attribute='name') | join(', ') }}.{% endif %}
        {% if silent %}Silent: {{ silent | map(attribute='name') | join(', ') }}.{% endif %}
```

A dashboard card listing the low devices with their area and battery type, so a glance tells you what to grab from the drawer:

```yaml
type: markdown
content: >-
  {% for d in state_attr('sensor.battery_sentinel', 'low') %}
  - **{{ d.name }}** ({{ d.area }}) {{ d.level }}%, {{ d.battery_type }}
  {% endfor %}
```

A leak-sensor case, which is the reason silent detection exists: a Third Reality water leak detector reports once a day and then, one week, simply stops. Its last reading still shows a full battery, so a percentage check says all is well while the sensor that is supposed to catch a flood is dead. A silent-mode sensor with the daily check set past the device's normal reporting gap flags it the next morning, and the leak sensor gets a fresh cell before it is ever needed.

A to-do consumer, dropping each low cell onto a shopping list so the right battery is on hand before the swap:

```yaml
trigger:
  - trigger: state
    entity_id: sensor.battery_sentinel
action:
  - repeat:
      for_each: "{{ state_attr('sensor.battery_sentinel', 'low') }}"
      sequence:
        - action: todo.add_item
          target:
            entity_id: todo.shopping
          data:
            item: "{{ repeat.item.name }} battery ({{ repeat.item.battery_type }})"
```

For more worked examples, detailed setup, and the fuller story behind each design choice, see the article: https://xeazy.com/battery-sentinel-blueprint/

## License

Copyright (C) 2026 James Lander, The Thinking Home (https://xeazy.com). This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.

## Changelog

| Version | Notes |
| --- | --- |
| 1.0.0-alpha | Initial build. One trigger-based template sensor classifying every monitored battery device into low, silent, or ok, with the offenders exposed as structured attributes. Low detection with a value-hysteresis band so a device at the threshold does not flap. Silent detection judged once a day against a lookback window, which spares legitimate daily reporters and catches the device that freezes at its last value without going unavailable. Low, silent, and both monitor modes. Include and exclude accepting entities, areas, devices, or labels, with exclude winning. Percentage and binary battery sensors supported; voltage staged for a later build. Under active testing on a live system. Increments to 1.0.0-beta, then 1.0.0, once it runs complete in place. |
