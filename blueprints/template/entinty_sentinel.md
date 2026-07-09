# Entity Sentinel (2.1.0 Beta)

A Home Assistant template blueprint that catches entities gone quiet: `unavailable`, `unknown`, `missing`, and, most important, `frozen` at their last value while the device behind them has silently died. One sensor, up to five tiers, each tier with its own targets and its own freeze window. Entity Sentinel gives you the count of troubled entities and names each one, ready to send as a notification, show on a dashboard, or speak aloud through a voice assistant.

Entity Sentinel comes from **The Thinking Home** at [xeazy.com](https://xeazy.com). The full design story and worked examples are in the [article](https://xeazy.com/battery-entity-sentinel-blueprints/), and the companion that turns this sensor into notifications and a live to-do list is [Sentinel Notify](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/automation/sentinel_notify.md).

**Current version: 2.1.0-beta.** The 2.x engine graduated through a 63-check adversarial simulation, two external code reviews (one of which surfaced a real bug, fixed before release), and live duty on the author's own system: nightly reboot cycles, real detections including a frozen sensor caught at exactly its tier's window, and planted-failure tests that shaped the engine itself: the freeze clock reads a device's own last-seen entity where one exists, so a restart or an integration re-publish cannot rewind it. Beta means the inputs and the output contract are settled; the engine keeps improving behind them, and the remaining risk is variety, more homes, more integrations, more hardware weirdness. If you hit something, the [community thread](https://xeazy.com/logbook/d/42-the-battery-entity-sentinel-blueprints) is where it gets found and fixed.

## How the Tiers Work

Give each tier the entities that share a reporting rhythm, and a freeze window matching that rhythm. One evaluation on one cadence judges every entity against its own tier's window, so a slow device can never false-flag on a fast window. Unavailable and unknown entities are flagged after one shared debounce, whatever their tier.

Three rules:

1. **A tier with no targets is skipped silently.** Most homes need only three, so leaving tiers 4 and 5 empty is normal, not a problem. All five empty is a loud `setup_error`.
2. **An entity may live in exactly one tier.** Tier priority runs 1 over 2 over 3 over 4 over 5: a duplicate stays monitored in its lowest-numbered tier. Any higher-numbered tier that also claims the entity drops it and names it in that tier's `status`, so you can either remove the entity from that tier's target or exclude it with `tier_N_exclude`.
3. **A tier watching nothing is a note, not an error.** Its `status` tells you why, an untagged label, or an exclude that covered everything, so a half-finished setup never looks broken.

## What It Detects

Each flagged entity carries the raw reason, exactly as classified:

- **`unavailable`**, the entity's own state is `unavailable`.
- **`unknown`**, the entity's own state is `unknown`.
- **`missing`**, the entity does not exist (renamed, removed, or never created).
- **`frozen`**, the entity still has a value but has not reported within its tier's window: it presents as healthy while the device behind it has gone quiet.
- **`never_reported`**, the entity exists but has never produced a reading.

Freeze is judged against the freshest report from any entity on the same device, so an entity that is quiet while a sibling on the same device is chatting is not frozen; the device is demonstrably alive. A motion sensor is a good example: its motion entity fires often, its signal strength updates constantly, and its battery percentage barely moves for months. Any one of them reporting keeps the whole device counted as alive, so the slow battery entity is never mistaken for frozen.

### The Freeze Clock, and Why last_seen Matters

The reasons split across two levels. `unavailable`, `unknown`, `missing`, and `never_reported` are judged at the **entity** level, read straight from the entity you named. `frozen` is judged at the **device** level, from the freshest report across the whole device, and *which clock* supplies that freshness matters.

Where a device exposes a **last-seen entity** (a sibling with `device_class: timestamp` and `last_seen` in its entity id, as Zigbee2MQTT and Z-Wave JS publish), the sensor reads that entity's **value** as the device's last report. That value is the protocol's own truth: it survives Home Assistant restarts and integration reconnects untouched. **Enable it on every device you watch for freezes.** It does not need a label or a tier; it only needs to be enabled. Many integrations ship it disabled by default, under the device's diagnostic entities: open the device page, show disabled entities, enable the last-seen entity, and it is on duty from that moment.

Where no last-seen entity exists, the sensor falls back to Home Assistant's `last_reported` metadata, and the fallback has a failure you must understand: **`last_reported` is reset by any event that re-pushes states.** Each reset rewinds the freeze clock to that moment, silently un-flagging a standing freeze and restarting detection from zero. Events that reset it:

1. **A Home Assistant restart.** Every entity in the system is re-pushed at startup, so every fallback freeze clock resets at every restart, scheduled or not.
2. **An integration reload.** Reloading an integration (or a template reload that recreates entities) re-pushes its entities' states.
3. **A connection loss and recovery.** An integration that reconnects and republishes, Zigbee2MQTT after losing its network-attached coordinator, an MQTT broker restart, a cloud integration re-syncing, stamps every one of its entities fresh.
4. **A network event that triggers any of the above.** A router reboot that briefly severs the path to a network-attached coordinator resets every clock behind it, even though Home Assistant itself never restarted.

The consequence, stated plainly: **on the fallback path, a freeze window longer than the time between resets can never fire.** A system that restarts nightly can never detect a fallback freeze with a window longer than a day; the clock is rewound before it matures. A frozen device is still caught eventually, detection restarts after each reset, and the `unavailable` path, which no republish erases, remains the backstop, but the fallback is the less reliable clock, and the last-seen entity is the fix.

The practical rules: enable `last_seen` wherever it exists, Zigbee and Z-Wave both offer it. For devices without one, a Wi-Fi presence sensor, for example, give the device a genuinely fast-reporting sibling to vouch for it (a light or signal-strength entity that updates constantly), size its freeze window shorter than your restart cadence, and let the unavailable detection cover the rest.

## Import the Blueprint

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Ftemplate%2Fentity_sentinel.yaml)

Or paste this URL into Settings, Automations & Scenes, Blueprints, Import Blueprint:

```
https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/template/entity_sentinel.yaml
```

Your imported copy is a snapshot; re-import to pick up newer releases, and check the version number in the blueprint description.

## Setting Up the Sensor

This is a **template** blueprint, so importing only registers it; there is no _Create_ button. It becomes a sensor through a `use_blueprint` entry, which you can add in one of two places.

**The quick way, straight in `configuration.yaml`.** Good for trying it out or if this is the only Sentinel you run. Add the `use_blueprint` block under a `template:` key in `configuration.yaml` (if you already have a `template:` key there, add this as another entry under it rather than making a second one), then restart. Skip to the sample configuration below.

**The organized way, a package file.** Better once you run more than one Sentinel, or if you like keeping related config together. A [package](https://www.home-assistant.io/docs/configuration/packages/) is just a file whose contents fold into your configuration at startup, so the same `use_blueprint` block lives in its own tidy file instead of crowding `configuration.yaml`. Set up once:

1. **Enable packages** (once, if you have not already). In `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

2. **Create the folder and file.** Make a `packages` folder inside your `config` folder if it does not exist, and create a file in it, for example `entity_sentinel.yaml`. The **`.yaml` extension is required**; a file without it is silently ignored, which is the single most common reason the sensor never appears.

3. **Paste a configuration** (the sample below is a working start). The `path:` is relative to `config/blueprints/template/`, so only the last part is used, exactly `TheThinkingHome/entity_sentinel.yaml`, whatever folder your file lives in.

4. **Restart Home Assistant once.** A brand-new package file registers at startup. After that, edits to the same file reload through Developer Tools, YAML, Reload Template Entities.

You will also need an **uptime sensor in timestamp mode**: Settings, Devices & Services, Add Integration, Uptime. The sensor uses it to know when Home Assistant started, which drives the startup grace.

## A Sample Configuration

Four tiers on a real system: three cadence classes plus a battery-liveness tier (which detects battery *entities gone silent*, a different question from [Battery Sentinel](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/battery_sentinel.md)'s low-level detection; run both). Tier 5 is simply absent, unused slots never appear:

```yaml
template:
  - use_blueprint:
      path: TheThinkingHome/entity_sentinel.yaml
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
        tier_3_freeze:
          hours: 36
        tier_4_name: Batteries
        tier_4_target:
          label_id:
            - battery_watch
        tier_4_freeze:
          hours: 36
```

Labels are the recommended targeting: create a label per rhythm (Settings, Areas & Zones, Labels), tag the entities whose silence you care about, and point each tier at its label. Adding a device to the watch later is then a tag, not a config edit.

## Upgrading from 1.x

If you run the 1.x pattern, one or more Entity Sentinel copies plus Dashboard Sentinel, the migration is one package edit. One thing to know first: re-importing this blueprint replaces your local 1.x copy, and the 1.x inputs no longer exist in 2.0, so re-import and migrate the package in the same sitting (or stay on 1.x by importing the frozen [entity_sentinel_1.0.5-beta.yaml](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/entity_sentinel_1.0.5-beta.yaml) instead):

1. **Replace the blocks.** Delete each 1.x `use_blueprint` block and the Dashboard Sentinel block, and add one 2.0 block. Each old copy becomes a tier: its `include_target` becomes that tier's `tier_N_target`, and its freeze window becomes `tier_N_freeze`. Your labels carry over untouched.
2. **Restart.** The new sensor appears under the `unique_id` you gave it.
3. **Repoint the consumers.** Anything that read the aggregator (dashboard cards, Sentinel Notify's source input) points at the new sensor instead. The `devices` list is shape-identical, so cards need only the entity_id swap; 2.0 additionally has `tier_error_count` where the aggregator had `source_error_count`.

The first evaluation after migration re-detects whatever was flagged before, same entities, same reasons, now annotated with their tier.

## Parameters

The five tier slots take the same four inputs each; N is 1 through 5.

| Parameter | Required | Default | What it does |
| --- | --- | --- | --- |
| `tier_N_name` | No | `Tier N` | The display name for the tier, carried on every flagged entry (`tier`) and in the per-tier counts. |
| `tier_N_target` | No | empty | Entities, labels, areas, or devices for this tier. Explicit entities and labels are watched precisely; areas and devices are swept to one representative entity per device. Empty means the tier is unused. |
| `tier_N_freeze` | No | 8h / 16h / 36h / 36h / 36h | How long a device in this tier may go without reporting anything before it counts as frozen, judged against the freshest report across the whole device. |
| `tier_N_exclude` | No | empty | Entities, labels, areas, or devices to leave out of **this tier only**. The tool for resolving a cross-tier duplicate without touching the label. |
| `exclude_target` | No | empty | Entities, labels, areas, or devices to leave out of **every** tier. Exclude always wins over any include. |
| `unavailable_debounce` | No | 3 minutes | How long an entity must stay unavailable or unknown before it is flagged, whatever its tier. |
| `scan_interval` | No | `/2` | How often the sensor re-evaluates, in minutes. The only latency knob: an entity is caught within one tick of its own tier's window expiring. |
| `startup_grace_seconds` | No | `240` | How long after Home Assistant starts to hold the unavailable list empty while the system and mesh come up. Freeze is not held (it cannot false-fire at startup). |
| `uptime_sensor` | Yes | `sensor.uptime` | The uptime sensor in TIMESTAMP mode. Missing or non-timestamp is a `setup_error`. |
| `refresh_button` | No | none | An optional `input_button` for an immediate re-evaluation. Share one button across Sentinels. |
| `sensor_name` | No | `Entity Sentinel` | The name of the created sensor. |
| `unique_id` | Yes | none | A unique id so the sensor is registered and editable in the interface. |
| `debug_enabled` | No | `false` | One summary log line per evaluation, with per-tier counts and every flagged entity. Needs a `logger` entry at info level to appear. |

## Attributes

Everything Entity Sentinel 1.x published, unchanged. Attributes are ordered so the useful summary sits on top and the reference fields below: `ok`, `error`, `total_monitored`, `unavailable_count`, `frozen_count`, `tier_error_count`, `devices` (each entry: `name`, `entity_id`, `area`, `tier`, `reason`, `since`, `last_seen`, `age`), `tiers`, then `sentinel_type`, `sentinel_version`, `uptime_status`, `boot_time`, `settled`, `last_evaluated`. The state is the merged flagged count, entity-id sorted.

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
3. **The `path:` is wrong.** It must be exactly `TheThinkingHome/entity_sentinel.yaml`; it is relative to `config/blueprints/template/` and no other part of the path is read.
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
