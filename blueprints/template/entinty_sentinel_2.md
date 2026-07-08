# Entity Sentinel

A Home Assistant template blueprint that catches entities gone quiet: `unavailable`, `unknown`, `missing`, and, most important, `frozen` at their last value while the device behind them has silently died. One sensor, up to five tiers, each tier with its own targets and its own freeze window. Entity Sentinel gives you the count of troubled entities and names each one, ready to send as a notification, show on a dashboard, or speak aloud through a voice assistant.

Entity Sentinel comes from **The Thinking Home** at [xeazy.com](https://xeazy.com). The full design story and worked examples are in the [article](https://xeazy.com/battery-entity-sentinel-blueprints/), and the companion that turns this sensor into notifications and a live to-do list is [Sentinel Notify](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/automation/sentinel_notify.md).

## Why 2.0, and What It Replaces

The 1.x Entity Sentinel was limited to one freeze window: you told it how long silence was allowed to last, and it watched everything you gave it against that single clock. But a motion sensor that reports every few minutes and a leak sensor that speaks once a day do not share the same definition of "too quiet." The 1.x answer was one sensor per reporting rhythm, so most homes needed several. You build a list for Lively entities that can be considered 'gone quiet' at 8 hours, another for Quiet entities that take a little longer at 16, and another for those sensors that spend most of their lives sleeping and give it a freeze window of 36 hours. This worked, but had a repercussion: three sensors meant three separate flagged lists, and making these lists useful, on a dashboard card, in a notifier, or as one number, took an extra aggregation step. That step was [Dashboard Sentinel](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/dashboard_sentinel.md), a blueprint whose sole reason to exist was merging the copies back into one list, removing any duplicates, and ordering them so that they make sense.

2.0 moves the aggregation inside the detector. The tiers live in the sensor: each tier carries its own targets and its own freeze window, one evaluation judges every entity against its own tier's clock, and the output is one clean list, the same shape the aggregator used to produce. One package block replaces three blocks plus an aggregator. The configuration changes shape slightly, the window inputs become per-tier inputs, and the result is one sensor doing what two blueprints and multiple sensors did together.

**This is a breaking change.** But anything you built from it keeps working. 2.0 adds to the previous version's output rather than changing it, so Sentinel Notify, your dashboard cards, and anything else you built read the new sensor unchanged. The break is only in configuration: the package block is written differently, which is why this is 2.0 and not 1.6. The old line is preserved for anyone not ready to migrate: **the 1.x blueprint and its README remain downloadable, frozen at 1.0.5-beta**, as [entity_sentinel_1.0.5-beta.yaml](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/entity_sentinel_1.0.5-beta.yaml) and [entity_sentinel_1.0.5-beta.md](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/entity_sentinel_1.0.5-beta.md). Existing 1.x sensors keep working forever; they simply receive no further development. Dashboard Sentinel stays available for existing 1.x setups for now, but it is being retired: once 2.0 reaches stable, it goes away. New setups should use 2.0's tiers instead.

**Current version: 2.0.0-beta.** The 2.0 engine graduated through a 57-check adversarial simulation, two external code reviews, and live duty on the author's own system: a week of nightly reboot cycles, real detections including a frozen door sensor caught at exactly its tier's window, and a full cutover replacing the three-sensor-plus-aggregator pattern it supersedes. Beta means the engine is settled: no more structural changes, and the only remaining risk is variety, more homes, more integrations, more hardware weirdness. If you hit something, the [community thread](https://xeazy.com/logbook/d/42-the-battery-entity-sentinel-blueprints) is where it gets found and fixed.

## How the Tiers Work

Give each tier the entities that share a reporting rhythm, and a freeze window matching that rhythm. One evaluation on one cadence judges every entity against its own tier's window, so a slow device can never false-flag on a fast window. 

Three rules:

1. **A tier with no targets is skipped silently.** Most homes need only three, so leaving tiers 4 and 5 empty is normal, not a problem. All five empty is a loud `setup_error`.
2. **An entity may live in exactly one tier.** Tier priority runs 1 over 2 over 3 over 4 over 5: a duplicate stays monitored in its lowest-numbered tier. Any higher-numbered tier that also claims the entity drops it and names it in that tier's `status` attribute, so you can either remove the entity from that tier's target or exclude it with `tier_N_exclude` parameter.
3. **A tier watching nothing is a note, not an error.** Its `status` tells you why, an untagged label, or an exclude that covered everything, so a half-finished setup never looks broken.

## What It Detects

Each flagged entity carries the raw reason, exactly as classified:

- **`unavailable`**, the entity's own state is `unavailable`.
- **`unknown`**, the entity's own state is `unknown`.
- **`missing`**, the entity does not exist. It has been renamed, removed, or it never existed.
- **`frozen`**, the entity still has a value but has not reported within its tier's freeze window: it presents as healthy while the device behind it has gone quiet.
- **`never_reported`**, the entity exists but has never produced a reading.

Freeze is judged against the freshest report from any entity on the same device. An entity that is quiet while a sibling on the same device is chatting is not frozen; the device is demonstrably alive. A motion sensor is a good example: its motion entity fires often, its signal strength updates constantly, and its battery percentage barely moves. Any one of them reporting keeps the whole device counted as alive, so the slow battery entity is never mistaken for frozen.

## Import the Blueprint

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Ftemplate%2Fentity_sentinel_2.yaml)

Or paste this URL into Settings, Automations & Scenes, Blueprints, Import Blueprint:

```
https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/template/entity_sentinel_2.yaml
```

Your imported copy is a snapshot; re-import to pick up newer releases, and check the version number in the blueprint description.

## Setting Up the Sensor

This is a **template** blueprint, so importing only registers it in your configuration; there is no _Create_ button. It becomes a sensor through a `use_blueprint` entry, which you can add in one of two places.

**The quick way, straight in `configuration.yaml`.** Good for trying it out or if this is the only Sentinel you plan to run. Add the `use_blueprint` block under a `template:` key in `configuration.yaml` (if you already have a `template:` key, add this as another entry under it rather than making a second one), then restart. Skip to the sample configuration below.

**The organized way, a package file.** Better once you run more than one Sentinel, or if you like keeping related config together. A [package](https://www.home-assistant.io/docs/configuration/packages/) is just a file whose contents fold into your configuration at startup, so the same `use_blueprint` block lives in its own tidy file instead of crowding your `configuration.yaml`. Four steps:

1. **Enable packages** (once, if you have not already). In `configuration.yaml`:

```yaml
   homeassistant:
     packages: !include_dir_named packages
```

2. **Create the folder and file.** Make a `packages` folder inside your `config` folder if it does not exist, and create a file in it, for example `entity_sentinel.yaml`. The **`.yaml` extension is required**; a file without it is silently ignored, which is the single most common reason the sensor never appears.

3. **Build the package file.** Open the file you just created and give it a `template:` block holding a `use_blueprint:` entry for this sensor. (A `template:` block is a list, so a second Sentinel, a Battery Sentinel, say, is just another `use_blueprint:` entry alongside it.) This is where the sensor is defined: the `use_blueprint:` points at the imported blueprint by its `path:`, and everything under `input:` configures your tiers. Paste the sample configuration below as a working starting point and edit its tiers to match your labels. The `path:` value, if you imported using the link above, is exactly:

```yaml
   path: TheThinkingHome/entity_sentinel_2.yaml
```

4. **Restart Home Assistant.** A brand-new package file registers at startup. After that, edits to the same file reload through Developer Tools, YAML, Reload `Template Entities`.

You will also need an **uptime sensor in timestamp mode**: Settings, Devices & Services, Add Integration, Uptime. The Sentinel uses it to know when Home Assistant last started, which drives the startup grace.

## A Sample Configuration

Three tiers on a real system, three cadence classes, with the Sleepy tier doing double duty: alongside its once-a-day sensors it also carries the `battery_watch` label, so battery *entities gone silent* are caught on the same 36-hour window (a different question from [Battery Sentinel](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/battery_sentinel.md)'s low-charge detection; run both). Tiers 4 and 5 are simply absent, unused slots never appear. Note that a tier's target takes a list, so `tier_3_target` here watches two labels at once; add as many as share that tier's freeze window.

```yaml
template:
  - use_blueprint:
      path: TheThinkingHome/entity_sentinel_2.yaml
      input:
        sensor_name: Entity Sentinel
        unique_id: entity_sentinel
        uptime_sensor: sensor.uptime
        scan_interval: "/30"
        unavailable_debounce:
          minutes: 3
        startup_grace_seconds: 240
        refresh_button: input_button.entity_sentinel
        tier_1_name: Lively
        tier_1_target:
          label_id:
            - entity_watch_lively
        tier_1_freeze:
          hours: 8
        tier_2_name: Quiet
        tier_2_target:
          label_id:
            - entity_watch_quiet
        tier_2_freeze:
          hours: 16
        tier_3_name: Sleepy
        tier_3_target:
          label_id:
            - entity_watch_sleepy
            - battery_watch
        tier_3_freeze:
          hours: 36
```

**Labels are the recommended targeting:** create a label per rhythm (Settings, Areas & Zones, Labels), tag the entities whose silence you care about, and point each tier at its label. Labels are the recommended targeting: make a label per rhythm (Settings, Areas & Zones, Labels), tag the entities whose silence you care about, and point each tier at its label. Later, adding a new sensor to the watch is just applying the label to it, no package edit or restart needed.

## Upgrading from 1.x

If you are currently running the older 1.x pattern, one or more Entity Sentinel copies plus Dashboard Sentinel, the migration is one package edit:

1. **Re-import the blueprint** using the link above. Re-importing updates your existing Entity Sentinel blueprint to 2.0. Its inputs are different, so migrate the package in this same sitting; your existing sensors are in limbo until you do. (Staying on 1.x instead? Import the frozen [entity_sentinel_1.x.yaml](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/entity_sentinel_1.x.yaml) and leave your blocks as they are.)

2. **Fold your Sentinels into one block, each old copy becoming a tier.** In a single 2.0 `use_blueprint` block, turn each old Entity Sentinel into one tier: its `include_target` becomes that tier's `tier_N_target`, and its `freeze_lookback` becomes `tier_N_freeze`. A tier needs three things, a name (`tier_N_name`), a target (`tier_N_target`), and a freeze window (`tier_N_freeze`); order them by window, shortest first, so tier 1 is your liveliest. Your labels carry over untouched, so a Lively copy watching `entity_watch_lively` at 8 hours becomes `tier_1_target: entity_watch_lively` with `tier_1_freeze: 8 hours`. Then delete the old 1.x blocks and the Dashboard Sentinel block.

3. **Restart.** The new sensor appears under the `unique_id` you gave it.

4. **Repoint the consumers.** Anything that read the old aggregator, dashboard cards, Sentinel Notify's source input, points at the new sensor instead. The `devices` list is shape-identical, so cards need only the entity_id swap; 2.0 has `tier_error_count` where the aggregator had `source_error_count`.

The first evaluation after migration re-detects whatever was flagged before, same entities, same reasons, now annotated with their tier.

## Parameters

The five tier slots take the same four inputs each; N is 1 through 5.

| Parameter | Required | Default | What it does |
| --- | --- | --- | --- |
| `tier_N_name` | No | `Tier N` | The display name for the tier, carried on every flagged entry (`tier`) and in the per-tier counts. (Example: Quite, Lively, Sleepy) |
| `tier_N_target` | No | empty | Entities, labels, areas, or devices for this tier. Explicit entities and labels are watched precisely; areas and devices are swept to one representative entity per device. Empty means the tier is unused. |
| `tier_N_freeze` | No | 8h / 16h / 36h | How long a device in this tier may go without reporting before it counts as frozen, judged against the freshest report across the whole device. |
| `tier_N_exclude` | No | empty | Entities, labels, areas, or devices to leave out of **this tier only**. The tool for resolving a cross-tier duplicate without touching the label. |
| `exclude_target` | No | empty | Entities, labels, areas, or devices to leave out of **every** tier. Exclude always wins over any include. |
| `unavailable_debounce` | No | 3 minutes | How long an entity must stay unavailable or unknown before it is flagged, whatever its tier. |
| `scan_interval` | No | `/2` | How often the sensor re-evaluates, in minutes. The only latency knob: an entity is caught within one tick of its own tier's window expiring. `/2` is fine for testing; raise it for everyday running: `/30` is every thirty minutes, or `"0"` is once an hour on the hour. |
| `startup_grace_seconds` | No | `240` | How long after Home Assistant starts to hold the unavailable list empty while the system and mesh come up.|
| `uptime_sensor` | Yes | `sensor.uptime` | The uptime sensor in TIMESTAMP mode. Missing or non-timestamp is a `setup_error`. |
| `refresh_button` | No | none | An optional `input_button` for an immediate re-evaluation. Share one button across Sentinels. |
| `sensor_name` | No | `Entity Sentinel` | The name of the created sensor. |
| `unique_id` | Yes | none | A unique id so the sensor is registered and editable in the interface. |
| `debug_enabled` | No | `false` | Writes one summary line to the log per evaluation when turned on: the version, per-tier counts, and every flagged entity by id. Off by default. To see the lines, set this `true` and add Sentinel's logger at info level in `configuration.yaml`: under `logger:`, set `logs:` with `sentinel_entity: info` (then restart). Useful when a sensor is flagging something you did not expect, or when you are tuning a tier's freeze window and want to watch what it catches. |

## Attributes

Everything Entity Sentinel 1.x published is unchanged, and 2.0 adds to it. Attributes are ordered so the useful summary sits on top and the reference fields below: `ok`, `error`, `total_monitored`, `unavailable_count`, `frozen_count`, `tier_error_count`, `devices` (each entry: `name`, `entity_id`, `area`, `tier`, `reason`, `since`, `last_seen`, `age`), `tiers`, then `sentinel_type`, `sentinel_version`, `uptime_status`, `boot_time`, `settled`, `last_evaluated`. The state is the merged flagged count, entity-id sorted.

A full sensor with one entity flagged looks like this:

```yaml
ok: true
error: ""
total_monitored: 165
unavailable_count: 1
frozen_count: 0
tier_error_count: 0
devices:
  - name: Hallway Motion
    entity_id: binary_sensor.hallway_motion
    area: Hallway
    tier: Lively
    reason: frozen
    since: "2026-07-08T14:30:12.421265+00:00"
    last_seen: "2026-07-08T06:30:44.954624+00:00"
    age: 8 hours
tiers:
  - tier: Lively
    monitored: 37
    flagged: 1
    status: ""
  - tier: Quiet
    monitored: 29
    flagged: 0
    status: ""
  - tier: Sleepy
    monitored: 99
    flagged: 0
    status: ""
sentinel_type: entity
sentinel_version: 2.0.0-beta
uptime_status: "2026-07-08T09:45:17+00:00"
boot_time: "2026-07-08T09:45:17+00:00"
settled: true
last_evaluated: "2026-07-08T14:30:00.188736-05:00"
```

New in 2.0:

| Attribute | What it carries |
| --- | --- |
| `tier` (on each `devices` entry) | The name of the tier that flagged the entity. |
| `tiers` | One entry per active tier: `tier` (the tier's name), `monitored`, `flagged`, and a single `status` string. `status` is empty when the tier is healthy; otherwise it carries one message: a cross-tier duplicate (naming each entity), a target that resolved to nothing (nothing tagged yet), or a target whose own exclude removed every resolved entity. |
| `tier_error_count` | How many tiers have a hard error (a cross-tier duplicate). A benign `status` note (nothing tagged, or all excluded) does not count. Zero when the configuration is clean. |

## If the Sensor Does Not Appear

In order of likelihood:

1. **The package file is not named `*.yaml`.** A missing extension is silently ignored.
2. **Packages are not enabled.** Check the `homeassistant: packages:` include in `configuration.yaml`.
3. **The `path:` is wrong.** It must be exactly `TheThinkingHome/entity_sentinel_2.yaml`; it is relative to `config/blueprints/template/` and no other part of the path is read.
4. **No restart since the file was created.** A brand-new package file registers only at startup; reload covers edits to an existing file, not new files.
5. **The blueprint is not imported.** Check Settings, Automations & Scenes, Blueprints for Entity Sentinel 2.0.
6. **A YAML error in the package.** Run the configuration check (Developer Tools, YAML, Check Configuration) and read the error it names.

If the sensor exists but reports `setup_error`, read its `error` attribute: it names the fault (an uptime sensor that is missing or not in timestamp mode, or no tiers configured).

## Changelog

| Version | Notes |
| --- | --- |
| 2.0.0-beta | The graduation release, promoting 2.0 to the recommended Entity Sentinel. Banked behind it: a 57-check adversarial simulation suite (five-tier boundary sweeps, duplicate and exclude permutations, representative-election collisions, hostile names, scale, and the settled contract a notifier depends on), two external code reviews with every finding reproduced against the file before acting (one real counting bug found and fixed), and a week of live duty through nightly reboot cycles, real frozen and unavailable detections, and a full cutover replacing the three-sensor-plus-aggregator pattern. The engine is unchanged from 2.0.0-alpha.1.2. |
| 2.0.0-alpha.1.2 | Count bucketing fix. An entity whose state is the literal string `none` (a badly behaved custom integration quirk) was flagged in `devices` and counted in the state, but fell into neither `unavailable_count` nor `frozen_count`, so the counts did not sum to the state. `none` now buckets into `unavailable_count`. Found by an externally proposed test case; reproduced, fixed, and proven in the suite. |
| 2.0.0-alpha.1.1 | Attribute presentation only; no logic change. Each `tiers` entry shows `tier: <name>` instead of a slot number plus a separate `name:` field; the old `error` and `note` fields merged into a single `status` string, which also distinguishes a tier whose own exclude removed every resolved entity from one whose target resolved to nothing; attributes reordered so the useful summary sits on top. |
| 2.0.0-alpha.1 | The tiered rebuild. Up to five tiers in one sensor, each with its own targets and freeze window on one shared cadence and one shared unavailable debounce. Duplicates across tiers resolve to the lowest-numbered tier and are reported in the higher tier's status, with partial rendering. Output contract is a strict superset of 1.x. Supersedes the multi-sensor pattern and the Dashboard Sentinel aggregator for new builds. |

The 1.x line and its full history live in the frozen [entity_sentinel_1.0.5-beta.md](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/entity_sentinel_1.0.5-beta.md).

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>). This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.
