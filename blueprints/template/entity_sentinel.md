# Entity Sentinel (Alpha)

A Home Assistant template blueprint that watches your critical entities and reports any that have gone quiet, the device that dropped offline and the one frozen at its last value, as a sensor anything in your system can read.

> **Status.** Entity Sentinel is being generalized from a battery not-reporting sensor into a full entity liveness monitor. This document describes that target design: watch any entity, catch both unavailable and freeze. The version you can import today still runs the previous battery not-reporting engine (it watches battery devices and uses the `report_hour` and `lookback_hours` inputs). The import link and setup mechanics below are correct now. The two-mode behavior and the parameter schema described here are in active development and land in a later version. Until then, treat the design sections as the roadmap and the import as the interim engine.

A device going quiet is rarely announced. A Zigbee sensor drops off the mesh, an integration stops serving one entity, a device hangs and freezes at its last value while still showing a healthy state. None of these throws an error, and a value check sails right past the frozen one because the value still looks fine. Entity Sentinel asks the question that catches all of them: has this entity actually reported lately, and is it still present?

Its companion, **Battery Sentinel**, counts the batteries running low. Battery Sentinel answers "what is running low," Entity Sentinel answers "what stopped reporting." Build both for full coverage, or just the one you want. They are separate blueprints and do not depend on each other.

## Why a Sensor Instead of an Automation

Entity Sentinel produces a `sensor`, not a one-time action. A sensor has a state and attributes that persist, and anything in Home Assistant can read it. One sensor, many listeners:

- **A notification** when a critical sensor goes quiet, so you find out before you need it.
- **A dashboard card** listing every entity that is offline or frozen, with how long it has been quiet.
- **A gate on another automation**, holding back logic that leans on a sensor that has stopped reporting, so a stale reading never drives an action.
- **A health badge** for the whole house, one number that is zero when everything is reporting.
- **A voice summary** through Assist, asking what has dropped before you leave or before bed.

## What It Does

Entity Sentinel watches the entities you name and flags any that have stopped reporting, by either of two checks, chosen automatically. The state is the count of flagged entities; each one is listed in a `devices` attribute with the reason it was flagged.

It asks two questions, at two levels, because the right level is different for each:

- **Unavailable or unknown, asked at the entity level.** The entity you named has gone `unavailable` or `unknown`. This is universal, it works on every entity regardless of integration, and it is the only signal for WiFi and cloud devices that expose no timestamp. Because it is asked at the entity level, an integration dropping one specific entity is caught even when the rest of the device is fine. It reports after a short debounce, so a brief blip does not flag.
- **Freeze, asked at the device level.** The device behind the named entity has not reported anything within a lookback window, even though it may still show a healthy value. This is the frozen-at-its-last-value case a state check cannot see. It is asked at the device level on purpose: a single entity can sit unchanged for days and be perfectly healthy, but a live device always has something reporting, link quality, motion, a temperature tick, so the freshest report across the whole device is the honest heartbeat.

The two run together, picked per entity with no configuration. Unavailable always applies. Freeze applies when the device exposes something whose timestamp advances; if nothing on the device does, unavailable covers it alone. An entity can be flagged by both for different reasons.

- **Structured output.** Each flagged entity carries its name, area, the `reason` it was flagged, how long it has been in that state, and its last-seen time with a human-readable age.
- **A grace period after a restart.** A restart briefly reads much of the mesh as `unavailable` while it repopulates. For a short window after Home Assistant starts (`startup_grace_seconds`), flagging is held back so a restart never reports a false fleet-wide outage.
- **Scope by entity, label, area, or device**, with include and exclude, and exclude always wins.

This is an alpha, under active testing on a live system. The design may still change. Do not rely on it for anything load-bearing until it graduates.

Full write-up and worked examples: <https://xeazy.com/battery-sentinel-blueprint/>  Questions and discussion: <https://xeazy.com/logbook/>

## How It Decides

A watched entity is judged by both checks, and either one flags it.

The **unavailable** check reads the entity's own state. If it is `unavailable` or `unknown` and stays that way past the debounce, it is flagged with that reason. This is the precise, entity-level question: did this exact entity stop being served.

The **freeze** check reads timestamps, not states. It resolves the device behind the entity and takes the freshest report across all of that device's entities. If even the freshest is older than the lookback window, the device is frozen and the entity is flagged. Reading the whole device defeats the sticky-entity trap: a door contact shut for two days has not changed, but its link-quality or battery entity is still checking in, so the device reads alive. The freeze check asks "has anything on this device reported lately," which is only false when the device is genuinely silent.

What freeze reads is "did the device send anything," not "did its value change." A temperature sensor reporting the same reading every minute is alive; a freeze check that watched only value changes would wrongly flag it. Reading the report timestamp, not the value, is what keeps a steady-but-live device off the list.

## Import the Blueprint

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Ftemplate%2Fentity_sentinel.yaml)

Or paste this URL into Settings, Automations & Scenes, Blueprints, Import Blueprint:

```
https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/template/entity_sentinel.yaml
```

Importing only registers a blueprint. It does not create a sensor yet. After import, the file lands at `config/blueprints/template/TheThinkingHome/entity_sentinel.yaml`. Verify it is there, because you will need that path in the next step.

## Setting Up the Sensor

### The Catch with Template Blueprints

Automation blueprints get a friendly Create automation button right on the Blueprints page. Template blueprints do not. You will not find this in the Create Helper list, and that is expected, not a fault. Template blueprints become real entities through a short piece of YAML called `use_blueprint`. It takes a path to the blueprint and a set of inputs:

```yaml
use_blueprint:
  path: TheThinkingHome/entity_sentinel.yaml
  input:
    sensor_name: Entity Sentinel
    unique_id: entity_sentinel
    include_target:
      label_id:
        - liveness_watch
```

`path` is relative to `config/blueprints/template/`, so it is the `TheThinkingHome` folder from the import step plus the filename. `unique_id` is required, and unlike Battery Sentinel, a scope is too: Entity Sentinel does not default to watching everything, since "every entity in the house" is thousands of them. You tell it what is critical.

That block has to sit under Home Assistant's `template:` configuration. The tidy way to do that, and the way that scales, is a package.

### What a Package Is, and How to Create One

A package is a single YAML file that holds a bundle of related configuration. Rather than scattering entities across your main files, you put everything for one feature into one file, and Home Assistant folds it into your overall configuration at startup. Packages are optional, but they keep related things together and easy to find.

If you have never used packages, switch them on once. In your `configuration.yaml`, add the two lines below. If you already have a `homeassistant:` section, just add the `packages:` line underneath it:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Then create a folder named `packages` next to your `configuration.yaml`. Every YAML file you drop in there is now a package.

Now make a package file, for example `packages/blueprint_entity_sentinel.yaml`, and put the `use_blueprint` block inside a `template:` list:

```yaml
template:
  - use_blueprint:
      path: TheThinkingHome/entity_sentinel.yaml
      input:
        sensor_name: Entity Sentinel
        unique_id: entity_sentinel
        freeze_lookback: "06:00:00"
        unavailable_debounce: "00:03:00"
        scan_interval: "/2"
        include_target:
          label_id:
            - liveness_watch
```

If you also build Battery Sentinel, the natural choice is to put both `use_blueprint` blocks in the same package file, each with its own `unique_id`.

### Load It

A brand-new package file needs a full Home Assistant restart to register the first time. After that, adding more `use_blueprint` blocks to the same file only needs a quick reload through Developer Tools, YAML, Reload Template Entities.

When it comes up you will have a sensor named after `sensor_name`. Watch it in Developer Tools, States. The state is the count, and the `devices` attribute lists the flagged entities.

## Scoping: Choosing What the Sensor Watches

Entity Sentinel requires a scope, set through `include_target` and `exclude_target`. Each accepts entities, labels, areas, or devices. There are two tiers, and the difference matters.

### Explicit Entities and Labels: Precise, Both Checks

Name entities directly, or tag them with a label such as `liveness_watch`, and each named entity is watched exactly as given, for both checks. This is the precise scope. Use it for the things you actually care about, your locks, leak sensors, the freezer probe, and it watches each one for both unavailable and freeze with no guessing.

```yaml
        include_target:
          label_id:
            - liveness_watch
```

A label is the natural way to run this: tag your critical entities once, and the sensor follows the tag.

### Areas and Devices: Convenient, Tuned for Freeze

Point the scope at an area or a device, and the sensor sweeps it. Because an area or device holds many entities, a sweep does not watch all of them. It picks one representative entity per device, the behaviorally meaningful one, chosen by a priority order: occupancy and motion first, then door, window, opening, garage, and lock, then moisture, smoke, gas, and carbon monoxide, then temperature and humidity, then connectivity, and battery last. A device with no such entity is skipped.

This is tuned for freeze, and it is correct for freeze, because freeze resolves to the device anyway, the representative is just the anchor. For the unavailable check it is approximate: the sweep watches the representative's availability as a stand-in for the device, so if the integration drops a different entity on that device while the representative stays present, the sweep does not see it.

**The rule of thumb:** use an area or device scope for broad freeze coverage of a room or a device. For precise unavailable coverage of specific entities, name them in a list or tag them with a label, which are watched exactly as given.

```yaml
        include_target:
          area_id:
            - nursery
```

### Exclude Always Wins

`exclude_target` takes the same kinds of targets and removes them from whatever the include produced. Exclude beats include every time, so it is how you trim a broad area sweep without abandoning it for a hand-built list:

```yaml
        exclude_target:
          entity_id:
            - sensor.nursery_humidity
```

## What the Sensor Looks Like

With one entity unavailable and one device frozen:

```yaml
sensor.entity_sentinel
State: 2

total_monitored: 14
last_evaluated: '2026-06-24T10:15:00-05:00'
devices:
  - name: Front Door Lock
    entity_id: lock.front_door
    area: Entry
    reason: unavailable
    since: '2026-06-24T10:02:00-05:00'
    last_seen: '2026-06-24T10:02:00-05:00'
    age: 13 minutes
  - name: Greenhouse Temperature
    entity_id: sensor.greenhouse_temp
    area: Greenhouse
    reason: frozen
    since: '2026-06-24T03:40:00-05:00'
    last_seen: '2026-06-24T03:40:00-05:00'
    age: 6 hours
```

The state is the number you badge or trigger on. Each entry carries its `reason` (`unavailable`, `unknown`, or `frozen`), `since` (when it first went bad), and `last_seen` with a human-readable `age`. A consumer can tell newly-offline from still-offline by comparing `since` to the previous evaluation, no separate list needed. `total_monitored` reports how many entities fell within the scope.

## Parameters

The parameter schema below is the target for the liveness engine and is being finalized during the build. Input names and defaults may settle as the engine is written.

| Input | Required | Default | Accepted values | What it does |
| --- | --- | --- | --- | --- |
| `unique_id` | Yes | none | any short text id, distinct per sensor | Registers the entity so you can rename it or place it in an area. Reusing one collides two sensors. |
| `include_target` | Yes | empty | entities, labels, areas, or devices | Which entities to watch. Explicit entities and labels are watched precisely for both checks; areas and devices are swept, one representative entity per device, tuned for freeze. |
| `exclude_target` | No | empty | entities, labels, areas, or devices | Which to leave out. Exclude always wins over include. |
| `freeze_lookback` | No | `06:00:00` | a duration | How long a device may go without reporting anything before it counts as frozen. Set it to a little longer than the entity's normal reporting interval: minutes for an active sensor, hours or days for a quiet one. |
| `unavailable_debounce` | No | `00:03:00` | a duration | How long an entity must stay `unavailable` or `unknown` before it is flagged. Absorbs brief blips on flaky WiFi or cloud devices. |
| `scan_interval` | No | `/2` | `/1`, `/2`, `/3`, `/5`, `/10`, `/15`, `/30` | How often the sensor re-evaluates, in minutes. This is the only latency knob: an entity is caught within one tick. A short cadence keeps detection prompt. |
| `startup_grace_seconds` | No | `120` | 0 to 900 seconds | After Home Assistant starts, how long to hold flagging back while the mesh repopulates, so a restart does not report a false outage. Set to 0 to disable. |
| `sensor_name` | No | `Entity Sentinel` | any text | The friendly name, and what the entity id is built from. |
| `debug_enabled` | No | `false` | on or off | When on, writes one diagnostic line to the system log each evaluation, naming the flagged entities and why. |

## The Sticky-Entity Caveat

Freeze reads the freshest report across a whole device, which handles most sticky entities on its own: a quiet contact is vouched for by its livelier siblings. The caveat is a device whose every entity is sticky, a sensor in a room that nothing happens in, where even the freshest report can be old while the device is perfectly healthy. That device can read as frozen when it is not.

Three ways to handle it, in order of preference:

1. **Enable one always-advancing entity on the device.** Many integrations expose a signal that updates on every contact even when nothing else changes, Zigbee2MQTT's `last_seen` (off by default, turn it on), or a ZHA link-quality sensor. With one entity whose timestamp always moves, the device's heartbeat is always honest. This is the single best accuracy improvement.
2. **Widen the freeze lookback** for that scope, so a naturally quiet device is given more room before it counts as frozen.
3. **Drop the device from freeze scope** and let the unavailable check cover it, by excluding it or watching it only through a precise list.

## Changelog

| Version | Notes |
| --- | --- |
| 1.0.0-alpha.6 | Renamed from Battery Sentinel - Not Reporting to Entity Sentinel, ahead of the engine work, so the file, name, logger, and docs are settled before the two-mode liveness rewrite. Scaffolding rename only: the importable engine is still the battery not-reporting logic from alpha.5. The liveness behavior described in this document is in development. |
| 1.0.0-alpha.5 | A device with no report timestamp anywhere, one that has never reported, is now flagged rather than skipped, carrying `last_seen: null` and `age: never reported`. A device that has never reported is the strongest not-reporting case there is, and this also surfaces a typo'd or phantom entity id instead of silently ignoring it. |
| 1.0.0-alpha.4 | Area and device scopes re-filtered to `device_class: battery`, matching Battery Sentinel. An area sweeps in every entity it holds; without the filter, non-battery entities were treated as monitored. Explicit entity lists and labels stay trusted. |
| 1.0.0-alpha.3 | Removed the `scan_interval` input from the battery engine. The sensor fired once a day at `report_hour` plus on Home Assistant start, instead of waking on a minute interval and doing nothing on most wakes. |
| 1.0.0-alpha.2 | Judge liveness by the device, not the battery entity. A battery reading is sticky and may report only when the percentage changes, so the old per-entity check false-flagged healthy devices. The freshest report across all of a device's entities became the heartbeat. |
| 1.0.0-alpha | Initial build. Single-purpose stopped-reporting sensor split from the combined Battery Sentinel blueprint. Judged once a day against a lookback window, catching the device that freezes at its last value without going unavailable. Include and exclude by entity, area, device, or label, with exclude winning. Optional debug log line. Under active testing on a live system. |

## Putting the Signal to Work

Because the sensor is a count with a `devices` list, anything can read it.

A notification when a critical entity goes quiet, naming what and why:

```yaml
trigger:
  - trigger: state
    entity_id: sensor.entity_sentinel
action:
  - action: notify.mobile_app_yourphone
    data:
      title: Something stopped reporting
      message: >-
        {% for d in state_attr('sensor.entity_sentinel', 'devices') %}
        {{ d.name }} ({{ d.reason }}, {{ d.age }}).
        {% endfor %}
```

A dashboard card listing every quiet entity with its reason and how long it has been out:

```yaml
type: markdown
content: >-
  {% for d in state_attr('sensor.entity_sentinel', 'devices') %}
  - **{{ d.name }}** ({{ d.area }}) {{ d.reason }}, {{ d.age }}
  {% endfor %}
```

A gate that holds back an automation when the entity it relies on has gone quiet:

```yaml
condition:
  - condition: template
    value_template: >-
      {{ 'sensor.greenhouse_temp' not in
         (state_attr('sensor.entity_sentinel', 'devices')
          | map(attribute='entity_id') | list) }}
```

For more worked examples, detailed setup, and the fuller story behind each design choice, see the article: <https://xeazy.com/battery-sentinel-blueprint/>

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>). This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.

You will find the full YAML, this README, and the one-click import badge at the [Thinking Home blueprints repository](https://github.com/TheThinkingHome/Automations).
