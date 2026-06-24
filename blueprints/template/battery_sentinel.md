# Battery Sentinel

Two Home Assistant template blueprints that watch every battery device in the house and report, as sensors, which ones are running low and which ones have quietly stopped reporting.

Most battery blueprints check a percentage once a day and send a message. That catches the cell fading toward empty, but it misses the failure that actually strands you: the device that sits reading eighty percent for days because eighty was the last value it ever sent, while it is really dead and Home Assistant never marked it unavailable. A percentage check cannot see that, and a plain unavailable check cannot either.

Battery Sentinel splits the job across two single-purpose sensors so each reports cleanly:

- **Battery Sentinel - Low** counts the batteries at or below a threshold.
- **Battery Sentinel - Not Reporting** counts the batteries that stopped reporting, including the one frozen at a healthy-looking value.

Neither notifies on its own. Each is a signal: the state is a count, the flagged devices are listed in a `devices` attribute, and you decide what reads it, a notification, a dashboard card, a to-do entry, or any automation you write. Build the pair for full coverage, or just the one you want.

These are alpha builds, under active testing on a live system. The design may still change. Do not rely on them for anything load-bearing until they graduate.

Full write-up and worked examples: https://xeazy.com/battery-sentinel-blueprint/  Questions and discussion: https://xeazy.com/logbook/

## What You Get

Two `sensor` entities, each with its state as a count and the offenders listed in a `devices` attribute.

- **Battery Sentinel - Low.** A device counts as low when its battery percentage is at or below the threshold, or when a binary battery sensor reads on. A value-hysteresis band keeps a device hovering at the threshold from flapping in and out of the list: it stays low until it climbs above the threshold plus a margin, which for a battery only happens when a fresh cell is fitted.
- **Battery Sentinel - Not Reporting.** A device counts as not reporting when it has not sent an update within a lookback window, whatever value it is still showing. The check is judged once a day at an hour you choose rather than continuously, so a water leak sensor that legitimately reports once a day is not flagged for the hours it is quiet, while a genuinely dead or frozen device is caught.
- **Clean, structured output.** Each sensor exposes only its own attributes. The `devices` list holds objects carrying the device name, area, level or last-seen time, and battery type, so every consumer formats them its own way.
- **Include and exclude by label, area, device, or entity.** Leave include empty and the sensor watches every battery device-class entity automatically. Set a label to watch only the devices you tag. Exclude always wins, for phones, tablets, or a device you are deliberately running down.
- **Percentage and binary battery sensors** are supported now. Voltage-reporting devices are staged for a later build.

## Requirements

Home Assistant 2026.5.4 or newer. That is the version they are verified on. No custom integrations are required. If you run the Battery Notes integration, battery types are picked up where exposed; if you do not, the battery type field is simply blank.

## Step 1: Import the Blueprints

Each blueprint imports on its own. Use both for full coverage.

**Battery Sentinel - Low**, one click:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Ftemplate%2Fbattery_sentinel_low.yaml)

Or paste this URL into Settings, Automations & Scenes, Blueprints, Import Blueprint:

```
https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/template/battery_sentinel_low.yaml
```

**Battery Sentinel - Not Reporting**, one click:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Ftemplate%2Fbattery_sentinel_not_reporting.yaml)

Or paste this URL:

```
https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/template/battery_sentinel_not_reporting.yaml
```

Importing only registers a blueprint. It does not create a sensor yet. After import, the files land at `config/blueprints/template/TheThinkingHome/battery_sentinel_low.yaml` and `config/blueprints/template/TheThinkingHome/battery_sentinel_not_reporting.yaml`. Verify they are there, because you will need those paths in the next step.

## Step 2: The Catch with Template Blueprints

Automation blueprints get a friendly Create automation button right on the Blueprints page. Template blueprints do not. You will not find these in the Create Helper list, and that is expected, not a fault. Template blueprints become real entities through a short piece of YAML called `use_blueprint`. It takes a path to the blueprint and a set of inputs:

```yaml
use_blueprint:
  path: TheThinkingHome/battery_sentinel_low.yaml
  input:
    low_threshold: 20
    sensor_name: Battery Sentinel Low
    unique_id: battery_sentinel_low
```

`path` is relative to `config/blueprints/template/`, so it is the `TheThinkingHome` folder from Step 1 plus the filename. `unique_id` is the only required setting; the rest have sensible defaults. Every parameter is laid out in the reference below.

That block has to sit under Home Assistant's `template:` configuration. The tidy way to do that, and the way that scales, is a package.

## Step 3: What a Package Is, and How to Create One

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
        sensor_name: Battery Sentinel Low
        unique_id: battery_sentinel_low

  - use_blueprint:
      path: TheThinkingHome/battery_sentinel_not_reporting.yaml
      input:
        report_hour: 8
        lookback_hours: 28
        scan_interval: "/2"
        sensor_name: Battery Sentinel Not Reporting
        unique_id: battery_sentinel_not_reporting
```

To narrow either sensor, add an `include_target` with a label, area, device, or entity, and an `exclude_target` for anything to drop. If you already keep template entities in your `configuration.yaml` under a `template:` key, you can drop the `use_blueprint` blocks into that list instead. The package is simply the cleaner home once you have a few.

## Step 4: Load It

A brand-new package file needs a full Home Assistant restart to register the first time. After that, adding more `use_blueprint` blocks to the same file only needs a quick reload through Developer Tools, YAML, Reload Template Entities.

When it comes up you will have a sensor named after each `sensor_name`. Watch them in Developer Tools, States. The state is the count, and the `devices` attribute lists the flagged devices.

## What the Sensors Look Like

**Battery Sentinel Low**, with two cells below the threshold:

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

**Battery Sentinel Not Reporting**, with one device gone quiet:

```yaml
sensor.battery_sentinel_not_reporting
State: 1

total_monitored: 27
last_evaluated: '2026-06-24T10:15:00-05:00'
eval_date: '2026-06-24'
devices:
  - name: Guest Room Temperature
    entity_id: sensor.guest_room_temp_battery
    area: Guest Bedroom
    last_seen: '2026-06-21T03:14:08-05:00'
    age: 3 days
    battery_type: CR2032
```

The state is the single number you badge or trigger on. The `devices` list is there for any consumer that wants the detail. On the Not Reporting sensor, `eval_date` records the day the last check ran; the sensor reads it back to hold the once-a-day verdict between scans.

## Where This Fits

The most-used battery blueprint in the ecosystem checks a percentage on a schedule and notifies. It works, and for a small house it is enough. Battery Sentinel sits a step past it in two ways.

First, the Not Reporting sensor finds the silent failure, not just the low one. The device frozen at eighty percent that quietly died is the one that bites, because the number looks fine right up until you need the sensor and it is not there. A percentage check never sees it. Scoping a stale-timestamp check to battery devices catches it, and the once-a-day judging keeps honest daily reporters from crying wolf.

Second, these are sensors, not notifiers. A notification blueprint locks you into one message format and one delivery path. A sensor is read by anything: a notification, yes, but also a dashboard, a to-do list, a voice summary, a REST export, or an automation that only acts when the list is non-empty. The notification is the least of what the signal can drive.

## Parameters: Battery Sentinel - Low

| Input | Required | Default | Accepted values | What it does |
| --- | --- | --- | --- | --- |
| `unique_id` | Yes | none | any short text id, distinct per sensor | Registers the entity so you can rename it or place it in an area. Reusing one collides two sensors. |
| `low_threshold` | No | `20` | 1 to 100 | A battery percentage at or below this counts as low. Binary battery sensors reading `on` always count as low. |
| `low_clear_margin` | No | `2` | 0 to 25 | How far above the threshold a flagged device must climb before it clears. The hysteresis band that stops flapping. |
| `include_target` | No | empty | entities, areas, devices, or labels | Which devices to watch. Empty watches every battery device-class entity. A label here watches only tagged devices. |
| `exclude_target` | No | empty | entities, areas, devices, or labels | Which to leave out. Exclude always wins over include. |
| `scan_interval` | No | `/2` | `/1`, `/2`, `/3`, `/5`, `/10`, `/15`, `/30` | How often the sensor re-scans. In the alpha this is in minutes for fast testing; it reverts to hours for production. |
| `sensor_name` | No | `Battery Sentinel Low` | any text | The friendly name, and what the entity id is built from. |
| `debug_enabled` | No | `false` | on or off | When on, writes one diagnostic line to the system log each evaluation, naming the flagged devices. |

## Parameters: Battery Sentinel - Not Reporting

| Input | Required | Default | Accepted values | What it does |
| --- | --- | --- | --- | --- |
| `unique_id` | Yes | none | any short text id, distinct per sensor | Registers the entity so you can rename it or place it in an area. Reusing one collides two sensors. |
| `report_hour` | No | `8` | 0 to 23 | The hour of day the check is judged. Evaluated once per day at this hour, not continuously. |
| `lookback_hours` | No | `28` | 1 to 168 | How many hours back from the check hour a device must have reported to count as alive. Raise it if normal daily reporters keep getting flagged. |
| `include_target` | No | empty | entities, areas, devices, or labels | Which devices to watch. Empty watches every battery device-class entity. A label here watches only tagged devices. |
| `exclude_target` | No | empty | entities, areas, devices, or labels | Which to leave out. Exclude always wins over include. |
| `scan_interval` | No | `/2` | `/1`, `/2`, `/3`, `/5`, `/10`, `/15`, `/30` | How often the sensor wakes to look. In the alpha this is in minutes for fast testing; it reverts to hours for production. The once-a-day check runs independently of this. |
| `sensor_name` | No | `Battery Sentinel Not Reporting` | any text | The friendly name, and what the entity id is built from. |
| `debug_enabled` | No | `false` | on or off | When on, writes one diagnostic line to the system log each evaluation, covering the daily-check timing and naming the flagged devices. |

## A Small Note on Not Reporting

Not reporting does not mean the literal `unavailable` state. It means the device has not sent an update within the lookback window, whatever value it is still showing. That is the point: a device that froze at eighty percent still reads `80`, and an `unavailable` check sails right past it. Asking instead whether the device reported at all catches both the explicitly-unavailable device and the silently-frozen one in a single test.

The check is judged once a day at your chosen hour, against the lookback window, rather than on a rolling basis. A rolling twenty-four hour window would flag a once-a-day reporter every day it ran a little late. Judging once a day, with a lookback you can widen to twenty-eight hours or more, lets an honest daily reporter pass while a genuinely dead device is caught. The verdict is held until the next daily check, so it does not flap between scans.

## A Small Note on Hysteresis

A battery hovering right at the threshold reports noise: 20.1, 19.9, 20.1. Without a guard, that flaps a device in and out of the low list. Battery Sentinel - Low puts a deadband in the value. A device enters the list when it drops below the threshold, and clears only when it climbs above the threshold plus the margin. Because a battery does not recover on its own, the only thing that pushes a flagged device past the clear point is a fresh cell, which is exactly when you want it to clear.

## Example Use

A notification reading both sensors, only speaking when something needs attention:

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

A leak-sensor case, which is the reason the Not Reporting sensor exists: a Third Reality water leak detector reports once a day and then, one week, simply stops. Its last reading still shows a full battery, so a percentage check says all is well while the sensor that is supposed to catch a flood is dead. Battery Sentinel - Not Reporting flags it the next morning, and the leak sensor gets a fresh cell before it is ever needed.

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

For more worked examples, detailed setup, and the fuller story behind each design choice, see the article: https://xeazy.com/battery-sentinel-blueprint/

## License

Copyright (C) 2026 James Lander, The Thinking Home (https://xeazy.com). These blueprints are free software: you may use, modify, and redistribute them under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). They are provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt them, keep this copyright and license notice intact.

## Changelog

| Blueprint | Version | Notes |
| --- | --- | --- |
| Battery Sentinel - Low | 1.0.0-alpha | Initial build. Single-purpose low-battery sensor split from the combined Battery Sentinel blueprint, so the entity carries only low attributes. Value-hysteresis band so a device at the threshold does not flap. Include and exclude by entity, area, device, or label, with exclude winning. Percentage and binary battery sensors; voltage staged for later. Optional debug log line. Under active testing on a live system. |
| Battery Sentinel - Not Reporting | 1.0.0-alpha | Initial build. Single-purpose stopped-reporting sensor split from the combined Battery Sentinel blueprint. Judged once a day against a lookback window, sparing legitimate daily reporters and catching the device that freezes at its last value without going unavailable. The daily verdict is latched in the sensor's own attributes. Include and exclude by entity, area, device, or label, with exclude winning. Optional debug log line. Under active testing on a live system. |
