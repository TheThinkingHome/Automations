# Entity Sentinel (Beta)

A Home Assistant template blueprint that watches your critical entities and reports any that have gone quiet, the device that dropped offline and the one frozen at its last value, as a sensor anything in your system can read.

> **Status.** Entity Sentinel is in beta. The two-mode engine described here is the engine you import today. It is an advanced blueprint: more capable than most, and more demanding to set up. It requires the uptime integration and a deliberately chosen grace period (both covered below). The power is real, and so is the configuration it asks of you. If you want a simpler starting point, build it against a small explicit list first, then widen.

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
- **A grace period after a restart.** A restart briefly reads much of the mesh as `unavailable` while it repopulates. For a window after Home Assistant starts (`startup_grace_seconds`, measured from the uptime sensor), flagging is held back so a restart never reports a false fleet-wide outage.
- **Scope by entity, label, area, or device**, with include and exclude, and exclude always wins.

Full write-up and worked examples: <https://xeazy.com/battery-entity-sentinel-blueprint/>  Questions and discussion: <https://xeazy.com/logbook/>

## How It Decides

A watched entity is judged by both checks, and either one flags it.

The **unavailable** check reads the entity's own state. If it is `unavailable` or `unknown` and stays that way past the debounce, it is flagged with that reason. This is the precise, entity-level question: did this exact entity stop being served.

The **freeze** check reads timestamps, not states. It resolves the device behind the entity and takes the freshest report across all of that device's entities. If even the freshest is older than the lookback window, the device is frozen and the entity is flagged. Reading the whole device defeats the sticky-entity trap: a door contact shut for two days has not changed, but its link-quality or battery entity is still checking in, so the device reads alive. The freeze check asks "has anything on this device reported lately," which is only false when the device is genuinely silent.

What freeze reads is "did the device send anything," not "did its value change." A temperature sensor reporting the same reading every minute is alive; a freeze check that watched only value changes would wrongly flag it. Reading the report timestamp, not the value, is what keeps a steady-but-live device off the list.

## Before You Start: the Uptime Requirement

Entity Sentinel measures its startup grace from a Home Assistant uptime sensor, so it needs one, in timestamp mode. This is the one hard dependency the blueprint adds, and it exists for a good reason: it is the reliable way to know how long Home Assistant has been running, which is what the grace period depends on.

Add the **Uptime** integration if you do not have it: Settings, Devices & Services, Add Integration, Uptime. It creates `sensor.uptime`, whose state is a timestamp like `2026-06-26T14:56:59+00:00` (the moment Home Assistant started). That timestamp form is what the blueprint needs.

If the uptime sensor is missing, or is set to a numeric mode instead of a timestamp, the Sentinel does not guess or limp along. It reports a `setup_error` state with a message telling you exactly what to fix, and it watches nothing until you fix it. This is deliberate: a monitor that is itself misconfigured should say so loudly, not fail quietly. See the `setup_error` details under the output section.

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
    uptime_sensor: sensor.uptime
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
        uptime_sensor: sensor.uptime
        freeze_lookback:
          hours: 6
        unavailable_debounce:
          minutes: 3
        scan_interval: "/2"
        startup_grace_seconds: 240
        include_target:
          label_id:
            - liveness_watch
```

If you also build Battery Sentinel, the natural choice is to put both `use_blueprint` blocks in the same package file, each with its own `unique_id`.

### Load It

A brand-new package file needs a full Home Assistant restart to register the first time. After that, adding more `use_blueprint` blocks to the same file only needs a quick reload through Developer Tools, YAML, Reload Template Entities.

When it comes up you will have a sensor named after `sensor_name`. Watch it in Developer Tools, States. The state is the count, and the `devices` attribute lists the flagged entities. If you see `setup_error` instead of a number, read the `error` attribute, it almost always means the uptime sensor needs attention.

## Setting the Grace Period

`startup_grace_seconds` is the one number on this blueprint you should set deliberately rather than leave at its default. It is the window after Home Assistant starts during which flagging is held back, so a restart, while the system and the mesh come up, does not report a false outage.

Because the grace is measured from the uptime sensor (the moment the Home Assistant process started), it has to cover your hub's **full** startup, not just part of it:

1. **Boot time.** How long from process start until Home Assistant and its integrations are up.
2. **Mesh settle time.** How long after that until your Zigbee or Z-Wave devices finish re-routing and report in.
3. **A margin** on top, so a slightly slow morning does not slip past the window.

Watch your own startup once and add the pieces up. A fast mini-PC may settle in three to four minutes; a slower hub with a large Z-Wave network may need five or six. The default is `240` (four minutes), a reasonable starting point for a small-to-medium system, but it is a starting point, not a universal answer. Set it too short and a restart can flash a brief false outage before the mesh finishes; set it generously and the only cost is that a genuine outage in the first few minutes after a restart waits until the window closes to show.

## Scoping: Choosing What the Sensor Watches

Entity Sentinel requires a scope, set through `include_target` and `exclude_target`. Each accepts entities, labels, areas, or devices. There are two tiers, and the difference matters.

### Explicit Entities and Labels: Precise, Both Checks

Name entities directly, or tag them with a label such as `liveness_watch`, and each named entity is watched exactly as given, for both checks. This is the precise scope. Use it for the things you actually care about, your locks, leak sensors, the freezer probe, and it watches each one for both unavailable and freeze with no guessing.

```yaml
        include_target:
          label_id:
            - liveness_watch
```

A label is the natural way to run this: tag your critical entities once, and the sensor follows the tag. Labels expand fully, a label on an entity, a label on a device, and a label on an area all resolve to the underlying entities, on both the include and the exclude side.

### Areas and Devices: Convenient, Tuned for Freeze

Point the scope at an area or a device, and the sensor sweeps it. Because an area or device holds many entities, a sweep does not watch all of them. It picks one representative entity per device, the behaviorally meaningful one. A device class with a clear behavioral meaning wins first, in this order: occupancy and motion, then door, window, opening, garage, and lock, then moisture, smoke, gas, and carbon monoxide, then temperature and humidity, then connectivity, then battery. If a device exposes none of those, the sweep falls back to the device's primary function, a light, switch, lock, cover, fan, climate, and so on, then to mains telemetry like power or energy, and finally to any ordinary entity on the device. So a smart plug, a bulb, or a relay is represented by its switch or light entity rather than skipped. The only devices skipped are those whose entire entity set is configuration or diagnostic (a device exposing nothing but a firmware-update entity, say), where there is genuinely nothing to monitor.

This is tuned for freeze, and it is correct for freeze, because freeze resolves to the device anyway, the representative is just the anchor. For the unavailable check it is approximate: the sweep watches the representative's availability as a stand-in for the device, so if the integration drops a different entity on that device while the representative stays present, the sweep does not see it.

**The area-sweep boundary to know:** an area or device sweep collects entities that belong to a hardware device. A standalone helper, a virtual boolean, a template sensor, or an unmapped integration that lives in an area but has no device behind it is not picked up by the sweep. Those have nothing to sweep to. If you need to watch one, name it explicitly or give it a label; do not rely on the area sweep to find it.

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

### Trusted Entities and Swept Devices Do Not Double-Count

If you both name an entity explicitly (or label it) and sweep the area or device it lives on, the entity is watched once, through the explicit name, not twice. The dedupe works at the device level: when a device is already covered by a trusted entity, the sweep does not add a second representative from the same device. So a dead device counts once, no matter how many of your scope rules happen to reach it.

## What the Sensor Looks Like

With one entity unavailable and one device frozen:

```yaml
sensor.entity_sentinel
State: 2

error: ''
uptime_status: '2026-06-24T06:00:00-05:00'
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

The state is the count of flagged entities, the number you badge or trigger on. Everything else lives in the attributes, so an automation, a dashboard card, or a notification can read the detail.

**What each field means:**

- **`state`** the number of entities currently flagged, or the string `setup_error` if the uptime sensor is missing or misconfigured (see below). Zero means everything in scope is reporting normally.
- **`ok`** a boolean, `true` when the sensor is functioning (whether or not it has flagged anything), `false` only when it is in `setup_error`. This is the safe field to gate downstream automations on; see the warning under "Putting the Signal to Work."
- **`error`** empty in normal operation. When `state` is `setup_error`, this carries the human-readable reason and what to fix.
- **`uptime_status`** what the engine read from the uptime sensor: the timestamp when valid, or the bad value (or `unknown`/`unavailable`) when not. This is a diagnostic, so you can confirm at a glance that the grace clock is healthy.
- **`total_monitored`** how many entities fell within the scope after include, exclude, and the per-device representative sweep. A scope problem (a mistyped label, a device with nothing to watch) shows up here as a count lower than you expect.
- **`unavailable_count`** how many of the flagged entities failed the entity-level check (`unavailable`, `unknown`, or `missing`).
- **`frozen_count`** how many failed the device-level freeze check. These two counts always sum to the state.
- **`devices`** the list of flagged entities, each with the fields below.
- **`boot_time`** the timestamp the grace window is measured from, taken from the uptime sensor. `null` during a `setup_error`.
- **`last_evaluated`** the timestamp of the most recent evaluation, so you can confirm the sensor is still ticking.

**Each flagged entity in `devices` carries:**

- **`name`** the entity's friendly name, or its device name, or the entity id, whichever is found first.
- **`entity_id`** the entity that was flagged.
- **`area`** its area, or `Unassigned` if it has none.
- **`reason`** why it was flagged. One of four values (see below).
- **`since`** when it first entered the bad state, read live from the entity's `last_changed` for the unavailable check, or the freshest device report for freeze. `null` for a missing or never-reported entity.
- **`last_seen`** the last time the entity or its device reported. `null` if it has never reported or does not exist.
- **`age`** a human-readable form of `last_seen`, for example `13 minutes` or `6 hours`. Reads `never reported` for a device with no report on record, and `unknown` for a missing entity.

**The `reason` values:**

- **`unavailable`** the entity exists and its own state is `unavailable`. The device dropped off the network, or the integration marked it unavailable. Flagged after the debounce.
- **`unknown`** the entity exists and its state is `unknown`. It is registered and present but has no usable value right now, often a sensor that has not reported since startup.
- **`missing`** the entity does not exist at all. A deleted device, a renamed entity, or a typo in the scope. This is how a stale reference in your configuration surfaces instead of failing silently.
- **`frozen`** the entity is present with a normal value, but its device has not reported anything within the freeze lookback. The device locked up while still looking healthy, the case a plain value check sails right past.
- **`never_reported`** the entity exists but has produced no report on record at all, no timestamp to measure. A freshly added entity that has not initialized, or one whose integration has never served it. `last_seen` is null and `age` reads `never reported`. This is an entity-level signal, distinct from `frozen` (which means a device that did report, then went silent).

The first three and `never_reported` come from the entity-level check and add into `unavailable_count`; `frozen` comes from the device-level check and is `frozen_count`. A consumer can tell a newly-offline entity from a long-dead one by comparing `since` to the previous evaluation, no separate list needed.

### The `setup_error` State

If the uptime sensor named in `uptime_sensor` is missing, unavailable, or not a timestamp, the sensor's state reads `setup_error` instead of a number, with a distinct `mdi:cog-off` icon. In that state it watches nothing, `total_monitored` is 0 and `devices` is empty, so it never produces a false reading off a broken clock. The `error` attribute tells you what went wrong:

- **uptime sensor not found or unavailable** the entity id in `uptime_sensor` does not exist, or is currently `unavailable`/`unknown`. Check the name, or add the Uptime integration.
- **uptime sensor is not a timestamp** the entity exists but its state is a number or some other non-timestamp value. Set the Uptime integration to its timestamp form.

The sensor heals itself the moment the uptime sensor is fixed; the next evaluation reads a valid timestamp and resumes normally. If you build a dashboard badge on the state, treat any non-numeric value as "needs attention," since `setup_error` is a string.

## Parameters

| Input | Required | Default | Accepted values | What it does |
| --- | --- | --- | --- | --- |
| `unique_id` | Yes | none | any short text id, distinct per sensor | Registers the entity so you can rename it or place it in an area. Reusing one collides two sensors. |
| `include_target` | Yes | empty | entities, labels, areas, or devices | Which entities to watch. Explicit entities and labels are watched precisely for both checks; areas and devices are swept, one representative entity per device, tuned for freeze. |
| `uptime_sensor` | Yes | `sensor.uptime` | a `sensor` in timestamp mode | The Home Assistant uptime sensor the startup grace is measured from. Must be a timestamp (its state is an ISO time, not a number). If missing or not a timestamp, the sensor reports `setup_error`. |
| `exclude_target` | No | empty | entities, labels, areas, or devices | Which to leave out. Exclude always wins over include. Labels expand on this side too. |
| `freeze_lookback` | No | `{hours: 6}` | a duration | How long a device may go without reporting anything before it counts as frozen. Set it to a little longer than the entity's normal reporting interval: minutes for an active sensor, hours or days for a quiet one. |
| `unavailable_debounce` | No | `{minutes: 3}` | a duration | How long an entity must stay `unavailable` or `unknown` before it is flagged. Absorbs brief blips on flaky WiFi or cloud devices. |
| `scan_interval` | No | `/2` | `/1`, `/2`, `/3`, `/5`, `/10`, `/15`, `/30` | How often the sensor re-evaluates, in minutes. This is the only latency knob: an entity is caught within one tick. A short cadence keeps detection prompt. |
| `startup_grace_seconds` | No | `240` | 0 to 1800 seconds | Measured from the uptime sensor, how long after Home Assistant starts to hold flagging back while the system and mesh come up. Must cover your full boot plus mesh-settle time plus a margin; see "Setting the Grace Period." Set to 0 to disable. |
| `refresh_button` | No | `input_button.none` | an `input_button` | An optional button; pressing it re-evaluates the sensor immediately. Point several Sentinel sensors at one button to refresh them all at once. It re-scans; it cannot force a device to report. |
| `sensor_name` | No | `Entity Sentinel` | any text | The friendly name, and what the entity id is built from. |
| `debug_enabled` | No | `false` | on or off | When on, writes one diagnostic line to the system log each evaluation, with the blueprint version, the state, the uptime status, and the flagged entities and why. |

## The Sticky-Entity Caveat

Freeze reads the freshest report across a whole device, which handles most sticky entities on its own: a quiet contact is vouched for by its livelier siblings. The caveat is a device whose every entity is sticky, a sensor in a room that nothing happens in, where even the freshest report can be old while the device is perfectly healthy. That device can read as frozen when it is not.

Three ways to handle it, in order of preference:

1. **Enable one always-advancing entity on the device.** Many integrations expose a signal that updates on every contact even when nothing else changes, Zigbee2MQTT's `last_seen` (off by default, turn it on), or a ZHA link-quality sensor. With one entity whose timestamp always moves, the device's heartbeat is always honest. This is the single best accuracy improvement.
2. **Widen the freeze lookback** for that scope, so a naturally quiet device is given more room before it counts as frozen.
3. **Drop the device from freeze scope** and let the unavailable check cover it, by excluding it or watching it only through a precise list.

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

A gate that holds back an automation when the entity it relies on has gone quiet. Note the `ok` check first: it confirms the Sentinel itself is healthy before trusting its list, so a misconfigured Sentinel does not wave the automation through:

```yaml
condition:
  - condition: template
    value_template: >-
      {{ is_state_attr('sensor.entity_sentinel', 'ok', true)
         and 'sensor.greenhouse_temp' not in
         (state_attr('sensor.entity_sentinel', 'devices')
          | map(attribute='entity_id') | list) }}
```

**A word on the state and downstream math.** In normal operation the state is a number, but if the uptime sensor is misconfigured the state becomes the string `setup_error`. Do not gate automations with `int()` math on the state, `{{ states('sensor.entity_sentinel') | int(0) == 0 }}` reads `setup_error` as 0 and would treat a broken Sentinel as "all healthy," firing automations on stale data. Gate on the `ok` attribute instead (`is_state_attr('sensor.entity_sentinel', 'ok', true)`), which is `true` only when the sensor is actually working, then read the count or the `devices` list.

For more worked examples, detailed setup, and the fuller story behind each design choice, see the article: <https://xeazy.com/battery-entity-sentinel-blueprint/>

## Changelog

| Version | Notes |
| --- | --- |
| 1.0.1-beta | Three fixes from the pre-public review. Area and device sweeps no longer silently skip a device that exposes no priority device class: the representative election now falls back through the device's primary domain (light, switch, lock, cover, and the rest), then mains telemetry, then any ordinary entity, so smart plugs, bulbs, and relays are caught. An entity that has genuinely never reported is now flagged `never_reported` rather than mislabeled `frozen`, counted with the entity-level failures. Added an `ok` boolean attribute, true only when the sensor is functioning, so downstream automations can gate on it rather than doing integer math on a state that may read `setup_error`. |
| 1.0.0-beta | First beta. The two-mode liveness engine: entity-level unavailable/unknown/missing with a duration debounce, device-level freeze against the freshest device report over a lookback, auto-routed per entity. Scope by entity, label, area, or device, with include and exclude (exclude wins, labels expand on both sides), and device-level dedupe so an entity reached by both a name and a sweep is watched once. Representative-per-device sweeping by a `device_class` priority order. Startup grace measured from a required uptime sensor (timestamp mode), so it is reliable across restarts and reloads and covers slow-booting hubs; a missing or non-timestamp uptime sensor yields a loud `setup_error` state with `error` and `uptime_status` attributes, and the sensor fails safe (watches nothing) until fixed. Timezone-safe grace math. Optional refresh button. Structured `devices` output with `reason`, `since`, `last_seen`, and a human-readable `age`. Hardened across an extensive adversarial test suite and proven on a live system, including a real power-loss outage, the device-dedupe collapse, the full grace cycle off the uptime clock, and the `setup_error` path end to end. |

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>). This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.

You will find the full YAML, this README, and the one-click import badge at the [Thinking Home blueprints repository](https://github.com/TheThinkingHome/Automations).
