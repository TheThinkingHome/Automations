# Battery Sentinel (Alpha)

A Home Assistant template blueprint that watches every battery device in the house and reports the ones running low, as a sensor anything in your system can read.

Battery monitoring in Home Assistant has looked the same for years. The common blueprints check each device's battery percentage on a schedule and push a notification to your phone. The notification is the entire product. You get a one-time alert and nothing else in your system can read the result.

The forums carry a steady run of feature requests that a notification-only design cannot reach: list the actual battery type so the alert tells you what to buy, filter selected battery devices with a `battery_watch` label, send the result somewhere other than a phone, surface the device that is currently offline rather than just low. These are not hard requests. They are simply out of reach for a blueprint whose only output is a notification.

**Battery Sentinel** takes a different starting point. Instead of building down from a notification, it builds up from a sensor.

Its companion, **Entity Sentinel**, watches any critical entity for going quiet, unavailable or frozen at its last value. Battery Sentinel answers "what is running low," Entity Sentinel answers "what stopped reporting." Build both for full coverage, or just the one you want. They are separate blueprints and do not depend on each other.

## Why a Sensor Instead of an Automation

Battery Sentinel produces a `sensor`, not a one-time action. A sensor has a state and attributes that persist, and anything in Home Assistant can read it. One sensor, many listeners:

- **A notification**, the obvious one. A companion automation blueprint for this is planned, so you get the polished alert without hand-writing the Jinja yourself.
- **A dashboard card** that lists the low devices with area and battery type, so a glance tells you what to grab from the drawer.
- **A to-do or shopping list**, each low cell dropped onto a list so the right battery is bought before the swap.
- **A voice summary** through Assist, asking which batteries need attention before you head out.
- **A gate on another automation**, logic that only runs when the list is empty or non-empty.

The notification is the least of what the signal can drive. It is just the first most people reach for.

## What It Does

Battery Sentinel counts the batteries at or below a threshold and lists each one, with its percentage, area, and battery type, in a `devices` attribute. The state is the count; the detail is in the attributes. You decide what reads it.

- **Percentage and binary, both handled.** A device counts as low when its battery percentage is at or below the threshold, or when a binary battery sensor reads `on`. A binary low battery sensor has no number to report, so its `level` comes through as `null`.
- **Hysteresis so it does not flap.** A value-hysteresis band keeps a device hovering at the threshold from bouncing in and out of the list. A flagged device stays low until it climbs above the threshold plus a margin, which for a battery only happens when a fresh cell is fitted.
- **Structured output.** The `devices` list holds objects carrying the device name, area, percentage level, and battery type if available. Every consumer can format them in its own way.
- **Offline batteries surfaced separately.** Any in-scope battery sitting at `unavailable` or `unknown`, an offline device, an integration that dropped an entity, a typo'd entity id, is not a low battery, so it never lands in the low list. Instead it is reported in its own `unavailable_entities` attribute, with a count in `unavailable_count`. The low state stays clean while you still get a window onto what is currently offline.
- **A grace period after a restart.** A momentary restart briefly reads most of the mesh as `unavailable` while it repopulates. For a short window after Home Assistant starts (`startup_grace_seconds`, default 120), the unavailable list is held empty so a restart never reports a false fleet-wide outage. The low count is never held; it evaluates live the whole time.
- **A manual refresh button.** Point an optional `input_button` at the sensor and a press re-evaluates it on the spot, handy right after you change a battery instead of waiting for the next scan. It re-scans; it cannot make a device report a fresh reading it has not sent yet.
- **Scope by label, area, device, or entity**, with include and exclude, and exclude always wins. Area and device scopes are filtered to real battery entities; labels and explicit entity lists are trusted as you curated them.

This is an alpha, under active testing on a live system. The design may still change. Do not rely on it for anything load-bearing until it graduates.

Full write-up and worked examples: <https://xeazy.com/battery-sentinel-blueprint/>  Questions and discussion: <https://xeazy.com/logbook/>

## Import the Blueprint

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Ftemplate%2Fbattery_sentinel.yaml)

Or paste this URL into Settings, Automations & Scenes, Blueprints, Import Blueprint:

```
https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/template/battery_sentinel.yaml
```

Importing only registers a blueprint. It does not create a sensor yet. After import, the file lands at `config/blueprints/template/TheThinkingHome/battery_sentinel.yaml`. Verify it is there, because you will need that path in the next step.

## Setting Up the Sensor

### The Catch with Template Blueprints

Automation blueprints get a friendly Create automation button right on the Blueprints page. Template blueprints do not. You will not find this in the Create Helper list, and that is expected, not a fault. Template blueprints become real entities through a short piece of YAML called `use_blueprint`. It takes a path to the blueprint and a set of inputs:

```yaml
use_blueprint:
  path: TheThinkingHome/battery_sentinel.yaml
  input:
    low_threshold: 20
    sensor_name: Battery Sentinel
    unique_id: battery_sentinel
```

`path` is relative to `config/blueprints/template/`, so it is the `TheThinkingHome` folder from the import step plus the filename. `unique_id` is the only required setting; the rest have sensible defaults. Every parameter is laid out in the reference table below.

That block has to sit under Home Assistant's `template:` configuration. The tidy way to do that, and the way that scales, is a package.

### What a Package Is, and How to Create One

A package is a single YAML file that holds a bundle of related configuration. Rather than scattering entities across your main files, you put everything for one feature into one file, and Home Assistant folds it into your overall configuration at startup. Packages are optional, but they keep related things together and easy to find.

If you have never used packages, switch them on once. In your `configuration.yaml`, add the two lines below. If you already have a `homeassistant:` section, just add the `packages:` line underneath it:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Then create a folder named `packages` next to your `configuration.yaml`. Every YAML file you drop in there is now a package.

Now make a package file, for example `packages/blueprint_battery_sentinel.yaml`, and put the `use_blueprint` block inside a `template:` list:

```yaml
template:
  - use_blueprint:
      path: TheThinkingHome/battery_sentinel.yaml
      input:
        low_threshold: 20
        low_clear_margin: 2
        scan_interval: "/2"
        sensor_name: Battery Sentinel
        unique_id: battery_sentinel
```

To narrow the sensor, add an `include_target` with a label, area, device, or entity, and an `exclude_target` for anything to drop. If you already keep template entities in your `configuration.yaml` under a `template:` key, you can drop the `use_blueprint` block into that list instead. The package is simply the cleaner home once you have a few. If you also build Entity Sentinel, the natural choice is to put both `use_blueprint` blocks in the same package file, each with its own `unique_id`.

### Load It

A brand-new package file needs a full Home Assistant restart to register the first time. After that, adding more `use_blueprint` blocks to the same file only needs a quick reload through Developer Tools, YAML, Reload Template Entities.

When it comes up you will have a sensor named after `sensor_name`. Watch it in Developer Tools, States. The state is the count, and the `devices` attribute lists the flagged devices.

## What the Sensor Looks Like

With two cells below the threshold and one battery currently offline:

```yaml
sensor.battery_sentinel
State: 2

total_monitored: 27
unavailable_count: 1
unavailable_entities:
  - name: Hall Motion
    entity_id: sensor.hall_motion_battery
    area: Hall
    state: unavailable
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

The state is the single number you badge or trigger on. The `devices` list is there for any consumer that wants the detail. `total_monitored` reports how many battery entities fell within the scope. A binary device shows `level: null`, since "on" carries no percentage. `unavailable_entities` is a separate list of in-scope batteries currently `unavailable` or `unknown`, each with its raw `state` (`unavailable`, `unknown`, or `missing` for an entity id that resolves to nothing), and `unavailable_count` is its length. These never affect the low state; they are a parallel view of what is offline.

## Scoping: Choosing What the Sensor Watches

The sensor decides which battery devices to watch through two inputs, `include_target` and `exclude_target`. Each accepts entities, areas, devices, or labels, and you can mix them.

### The Default: Watch Everything

Leave `include_target` out entirely and the sensor watches every battery device-class entity in your system. This is the widest scope and a fine place to start. On most systems it also catches phones, tablets, and watches, which carry battery entities through the companion app. Those charge nightly and are rarely what you want in a low-battery list, which is what `exclude_target` is for.

### Include by Entity

Name the exact entities you want, and the sensor watches precisely those. An explicit entity list is trusted: the blueprint watches what you named and does not second-guess it.

```yaml
        include_target:
          entity_id:
            - sensor.motion_master_battery
            - sensor.leak_kitchen_sink_battery
            - binary_sensor.door_garage_battery_low
```

### Exclude, and Exclude Always Wins

`exclude_target` takes the same kinds of targets and removes them from whatever the include produced. Exclude beats include every time, so it is how you drop the always-charged devices from a wide scope:

```yaml
        exclude_target:
          entity_id:
            - sensor.phone_james_battery_level
            - sensor.tablet_kitchen_battery_level
```

### Filter by Area or Device

Point the scope at an area or a device, and the sensor watches the battery entities within it. Areas and devices are **swept**: an area holds every entity assigned to it, and a device exposes many entities beyond its battery, so the blueprint keeps only the `device_class: battery` entities and drops the rest (signal strength, connectivity, blinds position, and so on).

```yaml
        include_target:
          area_id:
            - master
            - master_bath
```

### Filter by Label

A label lets you tag the exact devices you care about and watch only those, for example a `battery_watch` label on your critical infrastructure. There is one rule that decides whether this works at all:

**The label must live on the battery entity, not on the device.**

Home Assistant lets you apply a label to an entity, a device, or an area, and these are separate attachment points. This blueprint reads labels at the entity level. A label on a battery entity is found and watched. A label on a device is not, because the label sits on the device, not on the entities inside it.

Do this, label the battery entity itself:

```yaml
        include_target:
          label_id:
            - battery_watch
```

with the `battery_watch` label applied to `sensor.motion_master_battery`, `sensor.leak_kitchen_sink_battery`, and so on, the battery entities.

Not this: applying `battery_watch` to the Motion Master *device*. A device holds many entities, and the label on it is not seen by the sensor. That device is silently skipped, with no error and no warning.

The symptom to recognize: if a label-scoped sensor's `total_monitored` count looks lower than the number of devices you tagged, you almost certainly labeled devices instead of their battery entities. Move the label to the entity and the count corrects itself.

## Parameters

| Input | Required | Default | Accepted values | What it does |
| --- | --- | --- | --- | --- |
| `unique_id` | Yes | none | any short text id, distinct per sensor | Registers the entity so you can rename it or place it in an area. Reusing one collides two sensors. |
| `low_threshold` | No | `20` | 1 to 100 | A battery percentage at or below this counts as low. Binary battery sensors reading `on` always count as low. |
| `low_clear_margin` | No | `2` | 0 to 25 | How far above the threshold a flagged device must climb before it clears. The hysteresis band that stops flapping. |
| `include_target` | No | empty | entities, areas, devices, or labels | Which devices to watch. Empty watches every battery device-class entity. Areas and devices are filtered to batteries; labels and entity lists are trusted. |
| `exclude_target` | No | empty | entities, areas, devices, or labels | Which to leave out. Exclude always wins over include. |
| `scan_interval` | No | `/2` | `/1`, `/2`, `/3`, `/5`, `/10`, `/15`, `/30` | How often the sensor re-scans. In the alpha this is in minutes for fast testing; it reverts to hours for production. |
| `startup_grace_seconds` | No | `120` | 0 to 900 seconds | After Home Assistant starts, how long to hold the `unavailable_entities` list empty while the mesh repopulates, so a momentary restart does not report a false outage. The low count is never held. Set to 0 to disable. |
| `refresh_button` | No | none | an `input_button` entity | Optional. Press the named button to re-evaluate the sensor immediately, for example after changing a battery. It re-scans; it does not force a device to report. Several sensors can share one button. Leave unset to disable. |
| `sensor_name` | No | `Battery Sentinel` | any text | The friendly name, and what the entity id is built from. |
| `debug_enabled` | No | `false` | on or off | When on, writes one diagnostic line to the system log each evaluation, naming the flagged devices. |

## Changelog

| Version | Notes |
| --- | --- |
| 1.0.0-alpha.5 | Renamed from Battery Sentinel - Low to Battery Sentinel. The companion blueprint generalizes into Entity Sentinel, so the "Low" qualifier is no longer needed: this sensor is the battery state and level monitor. Name, file, `sensor_name` default, and debug logger renamed; detection unchanged. |
| 1.0.0-alpha.4 | Added a startup grace period for the offline list and an optional manual refresh button. For `startup_grace_seconds` after Home Assistant starts, `unavailable_entities` is held empty so a momentary restart no longer reports a false fleet-wide outage while the mesh repopulates; the low count is never held. The window is started only by the Home Assistant start event, never restarted by a scan or a button press. The `refresh_button` input re-evaluates the sensor on demand. Also fixed: an in-scope entity with no device (a typo'd or removed entity id) no longer raises a render error and instead appears in the offline list. |
| 1.0.0-alpha.3 | Added the `unavailable_count` and `unavailable_entities` attributes. Any in-scope entity at `unavailable`, `unknown`, or resolving to nothing is now listed separately, with its raw state, rather than silently dropped. The sensor state still counts only low batteries; an offline battery is a not-reporting problem, not a low one, so it is reported here instead of in the low list. |
| 1.0.0-alpha.2 | Area and device scopes are re-filtered to `device_class: battery`. Pointing the sensor at an area previously pulled in every entity there (signal strength, connectivity, blinds position) and read their numbers as battery levels. Explicit entity lists and labels stay trusted and unfiltered, since the user curates those. |
| 1.0.0-alpha | Initial build. Single-purpose low-battery sensor split from the combined Battery Sentinel blueprint, so the entity carries only low attributes. Value-hysteresis band so a device at the threshold does not flap. Include and exclude by entity, area, device, or label, with exclude winning. Percentage and binary battery sensors; voltage staged for later. Optional debug log line. Under active testing on a live system. |

## Putting the Signal to Work

Because the sensor is a count with a `devices` list, anything can read it.

A notification that speaks only when something needs attention:

```yaml
trigger:
  - trigger: state
    entity_id: sensor.battery_sentinel
action:
  - action: notify.mobile_app_yourphone
    data:
      title: Low batteries
      message: >-
        {% set low = state_attr('sensor.battery_sentinel', 'devices') %}
        {% if low %}{{ low | map(attribute='name') | join(', ') }}.{% endif %}
```

A dashboard card listing the low devices with area and battery type, so a glance tells you what to grab from the drawer:

```yaml
type: markdown
content: >-
  {% for d in state_attr('sensor.battery_sentinel', 'devices') %}
  - **{{ d.name }}** ({{ d.area }}) {{ d.level }}%, {{ d.battery_type }}
  {% endfor %}
```

A to-do consumer, dropping each low cell onto a shopping list so the right battery is on hand before the swap:

```yaml
trigger:
  - trigger: state
    entity_id: sensor.battery_sentinel
action:
  - repeat:
      for_each: "{{ state_attr('sensor.battery_sentinel', 'devices') }}"
      sequence:
        - action: todo.add_item
          target:
            entity_id: todo.shopping
          data:
            item: "{{ repeat.item.name }} battery ({{ repeat.item.battery_type }})"
```

For more worked examples, detailed setup, and the fuller story behind each design choice, see the article: <https://xeazy.com/battery-sentinel-blueprint/>

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>). This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.

You will find the full YAML, this README, and the one-click import badge at the [Thinking Home blueprints repository](https://github.com/TheThinkingHome/Automations).
