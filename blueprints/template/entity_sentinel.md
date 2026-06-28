# Entity Sentinel (Beta)

A Home Assistant template blueprint that watches a list of your critical entities and reports any that have gone quiet, the device that dropped offline and the one frozen at its last value, as a sensor anything in your system can read.

## What This Solves

A device going quiet is never announced. A Zigbee sensor drops off the mesh, an integration that stops serving a depreciated entity, a device hangs and freezes at its last value while still showing a healthy state. None of these throws an error, and a plain value check sails right past the frozen one, because the value still looks fine.

Home Assistant has no built-in answer for this, and the common approaches each miss a case: a simple `unavailable` check never catches the device that is actually "frozen" but still "present," and a value-change check wrongly flags a steady-but-healthy sensor that simply has not changed. What people keep asking for is one question that catches all of it: has this entity actually reported lately, and is it still alive?

That is the question Entity Sentinel asks and answers.

Its companion, **Battery Sentinel**, counts the batteries running low. Battery Sentinel answers "what is running low," Entity Sentinel answers "what stopped reporting." Build both for full coverage, or just the one you want. They are separate blueprints and do not depend on each other.

Plenty of blueprints already scan the whole house on a timer and list whatever is offline. Those are good at "set it up in two minutes and get a daily list of everything that is dead," and this does not replace them. Entity Sentinel is for the narrower request: it watches the specific entities that matter to you, it catches the frozen ones a state scan misses, and it works without polling the whole system on a schedule.

## Built on Request, and Why It Is a Beta

Every other blueprint from _The Thinking Home_ grew out of my own house. The sensors and automations behind them ran on my system for years before I shared them, so the reliability was already settled. My job generalized what already worked: made it configurable, documented it, smoothed the edges, and handed over something I already trusted.

This one is different. It did not run quietly for years in my walls. It was asked for, by the community, for a problem the existing tools cannot solve. So I built it: imagined, drafted, edited, reimagined, debugged, and tested as hard as one house and an adversarial test suite allows.

That is why this is a beta release. It is genuinely useful today, and I run it on my own system. But if you use it, you are also helping test and refine it. Real homes are more varied than any one setup, and the edge cases that matter most are the ones I did not imagine and cannot produce. If you hit something, the [community thread](https://xeazy.com/logbook/d/42-the-battery-entity-sentinel-blueprints) is where it gets found and fixed. That is the deal: I have done my best to build it, and the community's use is what will make it solid.

## Why a Sensor, Not an Automation

This builds a `sensor` which has a state and attributes that persist. Anything in Home Assistant can read it and that difference is is powerful. One sensor, many listeners:

- **A notification automation** when a critical sensor goes quiet, so you find out before you need it.
- **A dashboard card** listing every entity that is offline or frozen, with how long it has been quiet.
- **A gate on another automation**, holding back logic that leans on a sensor that has stopped reporting, so a stale reading never drives an action.
- **A health badge** for the whole house, one number that is zero when everything is reporting.
- **A voice summary** through Assist, asking what has dropped before you leave or before bed.

## What It Does

Entity Sentinel watches the entities you name and flags any that have stopped reporting, by either of two checks, chosen automatically. The state is the count of flagged entities; each is listed in a `devices` attribute with the reason it was flagged.

It asks two questions at two levels, because the right level differs for each:

- **Unavailable or unknown, at the entity level.** The named entity has gone `unavailable` or `unknown`. This is universal: it works on every entity, and it is the only signal for WiFi and cloud devices that expose no timestamp. Asked at the entity level, an integration dropping one specific entity is caught even when the rest of the device is fine. It reports after a short debounce, so a brief blip does not flag.
- **Freeze, at the device level.** The device behind the named entity has not reported anything within a lookback window, even though it may still show a healthy value. This is the frozen-at-its-last-value case a state check cannot see. It is asked at the device level on purpose: a single entity can sit unchanged for days and still be healthy, but a live device always has something reporting, link quality, motion, a temperature tick, so the freshest report across the whole device is the honest heartbeat.

The two run together, picked per entity with no configuration. Unavailable always applies. Freeze applies when the device exposes something whose timestamp advances; if nothing does, unavailable covers it alone. An entity can be flagged by both.

- **Structured output.** Each flagged entity carries its name, area, the `reason`, how long it has been in that state, and its last-seen time with a human-readable age.
- **A grace period after a restart**, measured from a required uptime sensor (see below), so a restart does not report a false fleet-wide outage.
- **A loud error if set up wrong.** A missing or misconfigured parameter yields a `setup_error` state with a distinct icon and a message naming the problem. An `ok` attribute lets automations tell a working sensor from a broken one.
- **Scope by entity, label, area, or device**, with include and exclude. Exclude is always given priority.

The full design, the reasoning behind each choice, and worked examples are in the article: <https://xeazy.com/battery-entity-sentinel-blueprints/>

## How It Decides

The **unavailable** check reads the entity's own state. If it is `unavailable` or `unknown` past the debounce, it is flagged. This is the precise, entity-level question: did this exact entity stop being served.

The **freeze** check reads timestamps, not states. It resolves the device behind the entity and takes the freshest report across all of that device's entities. If even the freshest is older than the lookback, the device is reported frozen. Reading the whole device solves the sticky-entity trap: a contact shut for two days has not changed, but its link-quality entity is still checking in, so the device reads alive. What freeze reads is "did the device send anything," not "did its value change," so a temperature sensor reporting the same reading every minute stays off the list.

## Before You Start: the Uptime Requirement

Entity Sentinel measures its startup grace from a Home Assistant **uptime sensor in timestamp mode**, so it needs one in place before it will work.

The startup grace needs to know how long the system has been up. The obvious way to track that, reading the sensor's own boot time from its attributes, is unreliable: Home Assistant has a long-standing bug ([#115585](https://github.com/home-assistant/core/issues/115585)) where a trigger-based template sensor sometimes cannot read its own attributes during evaluation. The uptime sensor sidesteps it: an independent entity, read fresh every evaluation, reliable across restarts and reloads.

Most systems already have it. If yours does not, add the **[Uptime Integration](https://www.home-assistant.io/integrations/uptime/)** (Settings, Devices & Services, Add Integration, Uptime). It creates `sensor.uptime`, whose state is a timestamp like `2026-06-26T14:56:59+00:00`. Confirm it is in **timestamp** mode, not a number of days.

If the uptime sensor is missing, unavailable, or not a timestamp, Entity Sentinel reports a `setup_error` state with a message naming the problem, watches nothing, and starts working the moment you fix it.

## Import the Blueprint

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Ftemplate%2Fentity_sentinel.yaml)

Or paste this URL into Settings, Automations & Scenes, Blueprints, Import Blueprint:

```
https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/template/entity_sentinel.yaml
```

Importing only registers a blueprint. It does not create a sensor yet. After import, the file lands at `config/blueprints/template/TheThinkingHome/entity_sentinel.yaml`. Verify it is there, you will need that path next.

## Setting Up the Sensor

### The Catch with Template Blueprints

Automation blueprints get a Create automation button on the Blueprints page. Template blueprints do not, and that is expected, not a fault. They become real entities through a short piece of YAML called `use_blueprint`, which takes a path and a set of inputs. Unlike Battery Sentinel, a scope is required: Entity Sentinel does not default to watching everything, since "every entity in the house" is hundreds. You tell it what is critical.

### What a Package Is, and How to Create One

A package is a single YAML file holding a bundle of related configurations that Home Assistant folds in at startup. Packages are optional, but they keep related things together.

If you have never used packages, switch them on once. In `configuration.yaml`, add the `packages:` line (under your existing `homeassistant:` section if you have one):

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Then create a folder named `packages` next to your `configuration.yaml`. Every YAML file you drop in there is a package.

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

If you also build Battery Sentinel, put both `use_blueprint` blocks in the same package file, each with its own `unique_id`.

### Load It

A brand-new package file needs a full Home Assistant restart to register the first time. After that, adding more blocks to the same file only needs a reload through Developer Tools, YAML, Reload Template Entities.

When it comes up you will have a sensor named after `sensor_name`. Watch it in Developer Tools, States. The state is the count; the `devices` attribute lists the flagged entities. If you see `setup_error` instead of a number, read the `error` attribute; it almost always means the uptime sensor needs attention.

## Parameters

| Input | Required | Default | Accepted values | What it does |
| --- | --- | --- | --- | --- |
| `unique_id` | Yes | none | any short text id, distinct per sensor | Registers the entity so you can rename it or place it in an area. Reusing one collides two sensors. |
| `include_target` | Yes | empty | entities, labels, areas, or devices | Which entities to watch. Explicit entities and labels are watched precisely for both checks; areas and devices are swept, one representative per device, tuned for freeze. |
| `uptime_sensor` | Yes | `sensor.uptime` | a `sensor` in timestamp mode | The uptime sensor the startup grace is measured from. Must be a timestamp. Missing or non-timestamp yields `setup_error`. See "Before You Start." |
| `exclude_target` | No | empty | entities, labels, areas, or devices | Which to leave out. Exclude always wins. Labels expand on this side too. |
| `freeze_lookback` | No | `{hours: 6}` | a duration | How long a device may go without reporting anything before it counts as frozen. Set it a little longer than the entity's normal reporting interval. |
| `unavailable_debounce` | No | `{minutes: 3}` | a duration | How long an entity must stay `unavailable` or `unknown` before it is flagged. Absorbs brief blips. |
| `scan_interval` | No | `/2` | `/1`, `/2`, `/3`, `/5`, `/10`, `/15`, `/30` | How often the sensor re-evaluates, in minutes. An entity is caught within one tick. |
| `startup_grace_seconds` | No | `240` | 0 to 1800 seconds | Measured from the uptime sensor, how long after Home Assistant starts to hold flagging back. Must cover your full boot plus mesh-settle plus a margin. See "Setting the Grace Period." |
| `refresh_button` | No | `input_button.none` | an `input_button` | Optional. Pressing it re-evaluates immediately. Point several Sentinels at one button to refresh them together. It re-scans; it cannot force a device to report. |
| `sensor_name` | No | `Entity Sentinel` | any text | The friendly name, and what the entity id is built from. |
| `debug_enabled` | No | `false` | on or off | When on, writes one diagnostic line to the system log each evaluation, with the version, state, uptime status, the flagged entities and why. |

## Setting the Grace Period

`startup_grace_seconds` is an input to set carefully because the right value depends on your hardware. Measured from the uptime sensor (the moment the Home Assistant process started), it has to cover your hub's full startup: boot time, plus the time your Zigbee or Z-Wave mesh takes to re-route and report in. This cannot be guessed, only deliberatly timed.

The default `240` (four minutes) suits a fast mini-PC with a stable mesh; a slower hub with a large Zigbee or Z-Wave network may need five or six. Set it too short and a restart can flash a brief false outage before the mesh finishes; set it generously and the only cost is that a genuine outage in the first few minutes after a restart waits until the window closes to show.

## Scoping: Choosing What the Sensor Watches

Entity Sentinel requires a scope, set through `include_target` and `exclude_target`. Each accepts entities, labels, areas, or devices. There are two tiers and understanding them is important.

**Explicit entities and labels are precise, and watched for both freeze and `unavailable`checks.** Name entities directly, or tag them with a label such as `entity_watch`, and each is watched exactly as given. Use this for the things you care about, your locks, leak sensors, the freezer probe. A label is the natural way to run this: label your critical entities once and the sensor follows the tag. Labels expand fully, on an entity, a device, or an area, on both the include and exclude side.

**Areas and devices are convenient, and tuned for freeze.** Point the scope at an area or device and the sensor sweeps it, picking one representative entity per device. A device class with a clear behavioral meaning wins first (occupancy, motion, door, window, lock, smoke, and so on), then the device's primary function (light, switch, lock, cover, and the rest), then telemetry, then any ordinary entity. So a smart plug or bulb is represented rather than skipped; only a device whose entire entity set is configuration, diagnostic, or event entities is skipped, since it offers no honest liveness signal. This is correct for freeze, which resolves to the device anyway. For the unavailable check it is approximate: the sweep watches the representative as a stand-in, so if the integration drops a different entity on that device, the sweep does not see it.

**Use an area or device scope for broad freeze coverage of a room; name entities or use a label for precise unavailable coverage.** Exclude always wins, so you can trim a broad sweep without abandoning it.

A standalone helper, virtual boolean, or template sensor with no device behind it is not picked up by an area sweep, there is nothing to sweep to. Name it explicitly or label it.

```yaml
        include_target:
          label_id:
            - entity_watch
        exclude_target:
          entity_id:
            - sensor.nursery_humidity
```

## What the Sensor Looks Like

With one entity unavailable and one device frozen:

```yaml
sensor.entity_sentinel
State: 2

ok: true
error: ''
uptime_status: '2026-06-24T06:00:00-05:00'
total_monitored: 14
unavailable_count: 1
frozen_count: 1
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

The state is the count of flagged entities, or the string `setup_error` if misconfigured. The attributes carry the detail:

- **`state`** the count of flagged entities, or `setup_error`.
- **`ok`** `true` when the sensor is functioning, `false` only in `setup_error`. The safe field to gate automations on.
- **`error`** empty normally; carries the reason and fix when in `setup_error`.
- **`uptime_status`** what the engine read from the uptime sensor. A diagnostic.
- **`total_monitored`** how many entities fell within the scope after include, exclude, and the per-device sweep.
- **`unavailable_count`** / **`frozen_count`** how many failed each check; they sum to the state.
- **`devices`** the flagged entities, each with the fields below.
- **`boot_time`** the timestamp the grace is measured from, null in `setup_error`.

**Each flagged entity carries** `name`, `entity_id`, `area`, `reason`, `since` (when it entered the bad state), `last_seen`, and a human-readable `age`.

**The `reason` values:**

- **`unavailable`** the entity exists and its state is `unavailable`. Flagged after the debounce.
- **`unknown`** the entity exists and its state is `unknown`. Present but no usable value, often a sensor that has not reported since startup.
- **`missing`** the entity does not exist at all. A deleted device, a renamed entity, or a typo in the scope.
- **`frozen`** the entity is present with a normal value, but its device has not reported within the freeze lookback. The locked-up-but-healthy case.
- **`never_reported`** the entity exists but has no report on record at all. A freshly added entity, or one its integration has never served.

The first three and `never_reported` come from the entity-level check (`unavailable_count`); `frozen` is the device-level check (`frozen_count`).

### The `setup_error` State

If the uptime sensor is missing, unavailable, or not a timestamp, the state reads `setup_error` with a distinct `mdi:cog-off` icon. In that state the sensor does not function, so it never produces a false reading off a broken clock. The `error` attribute names the cause, the entity not found, or the value not a timestamp, and the sensor heals itself the moment the uptime sensor is fixed. If you build a dashboard badge on the state, treat any non-numeric value as "needs attention."

## The Sticky-Entity Caveat

Freeze reads the freshest report across a whole device, which handles most sticky entities on its own: a quiet contact is vouched for by its livelier siblings. The caveat is a device whose every entity is sticky, a sensor in a room where nothing happens, where even the freshest report can be old while the device is healthy. That device can read as frozen when it is not.

The best fix is to enable one always-advancing entity on the device, Zigbee2MQTT's `last_seen` (off by default, turn it on) or a ZHA link-quality sensor, so the device's heartbeat is always honest. Otherwise, widen the freeze lookback for that scope, or drop the device from freeze scope and let the unavailable check cover it.

Some battery powered Zigbee and Z-Wave devices wake very infrequently if they have nothing new to report. For instance, a leak sensor that is dry. Some sensors may only report on a 25 hours schedule to extend battery life. Choose the `freeze_lookback` parameter carefully so that the sensor doesn't trigger on false positives.

## Putting the Signal to Work

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

A gate that holds an automation when the entity it relies on has gone quiet. The `ok` check comes first, so a misconfigured Sentinel does not wave the automation through:

```yaml
condition:
  - condition: template
    value_template: >-
      {{ is_state_attr('sensor.entity_sentinel', 'ok', true)
         and 'sensor.greenhouse_temp' not in
         (state_attr('sensor.entity_sentinel', 'devices')
          | map(attribute='entity_id') | list) }}
```

Do not gate on `int()` math alone: `int('setup_error')` evaluates to 0, which would read a broken Sentinel as "all healthy." Always check `ok` first.

More worked examples are in the article: <https://xeazy.com/battery-entity-sentinel-blueprints/>

## Changelog

| Version | Notes |
| --- | --- |
| 1.0.2-beta | Excluded event entities from the area and device sweep. A device whose only entities are event entities (an NSPanel exposing just touch and button entities, for example) was flagged with a false `unknown` because an event entity sits at `unknown` until it fires. Event entities are not a usable liveness signal, so the sweep now skips them; a device that offers nothing else is skipped, and an event entity never displaces a real sensor when one is present. |
| 1.0.1-beta | Three fixes from the pre-public review. Area and device sweeps no longer silently skip a device that exposes no priority device class: the representative election falls back through the device's primary domain (light, switch, lock, cover, and the rest), then mains telemetry, then any ordinary entity, so smart plugs, bulbs, and relays are caught. An entity that has genuinely never reported is now flagged `never_reported` rather than mislabeled `frozen`. Added an `ok` boolean so automations can gate on it rather than doing integer math on a state that may read `setup_error`. |
| 1.0.0-beta | First beta. The two-mode liveness engine: entity-level unavailable/unknown/missing with a duration debounce, device-level freeze against the freshest device report over a lookback, auto-routed per entity. Scope by entity, label, area, or device, with include and exclude (exclude wins, labels expand on both sides), and device-level dedupe so an entity reached by both a name and a sweep is watched once. Representative-per-device sweeping by a `device_class` priority order. Startup grace measured from a required uptime sensor (timestamp mode); a missing or non-timestamp uptime sensor yields a loud `setup_error` state with `error` and `uptime_status` attributes, and the sensor fails safe until fixed. Timezone-safe grace math. Optional refresh button. Structured `devices` output with `reason`, `since`, `last_seen`, and a human-readable `age`. Hardened across an extensive adversarial test suite and proven on a live system, including a real power-loss outage, the device-dedupe collapse, the full grace cycle off the uptime clock, and the `setup_error` path end to end. |

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>). This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.

You will find the full YAML, this README, and the one-click import badge at the [Thinking Home blueprints repository](https://github.com/TheThinkingHome/Automations).
