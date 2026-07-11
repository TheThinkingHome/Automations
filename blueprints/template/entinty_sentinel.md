# Entity Sentinel (2.2.0 Beta)

A Home Assistant template blueprint that catches entities gone quiet: `unavailable`, `unknown`, `missing`, and, most important, `frozen` at their last value while the device behind them has silently died. One sensor, up to five tiers, each tier with its own targets and its own freeze window. Entity Sentinel gives you the count of troubled entities and names each one, ready to send as a notification, show on a dashboard, or speak aloud through a voice assistant.

Entity Sentinel comes from **The Thinking Home** at [xeazy.com](https://xeazy.com). The full design story and worked examples are in the [article](https://xeazy.com/battery-entity-sentinel-blueprints/). It has a sister detector, [Battery Sentinel](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/battery_sentinel.md), and a companion built for both: [Sentinel Notify](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/automation/sentinel_notify.md) reads this sensor and turns it into change-aware phone notifications and a live to-do list you check off as you fix things. Entity Sentinel detects; Sentinel Notify tells you. They are built to work together, and most installations will want both.

**Current version: 2.2.0-beta.** The 2.x engine graduated through a 63-check adversarial simulation, two external code reviews (one of which surfaced a real bug, fixed before release), and live duty on the author's own system: nightly reboot cycles, real detections including a frozen sensor caught at exactly its tier's window, and planted-failure tests that shaped the engine itself. Beta means the inputs and the output contract are settled; the engine keeps improving behind them, and the remaining risk is variety, more homes, more integrations, more hardware weirdness. If you hit something, the [community thread](https://xeazy.com/logbook/d/42-the-battery-entity-sentinel-blueprints) is where it gets found and fixed.

## Why It Exists

There is a failure nothing else catches cleanly: a device that stops reporting but keeps showing its last value. It does not go `unavailable`. It throws no error. Home Assistant still displays the old reading, every automation downstream keeps trusting it, and the house quietly stops doing what you built it to do. Catching a device that drops offline is easy, a state check finds it. A frozen one passes every check you would think to write, because its state is still a perfectly valid value.

Entity Sentinel catches the freeze, and the ordinary failures alongside it. You name the entities you care about, and it watches each one: the `unavailable`, `unknown`, and `missing` you can already catch, and the `frozen` ones you currently cannot. It publishes one clean sensor: the count as its state, and a list naming every troubled entity, the reason it tripped, and how long it has been silent.

## How It Works, Just Enough

**Tiers.** Devices do not share a definition of "too quiet." A motion sensor reports every few minutes; a leak sensor speaks once a day and is perfectly healthy doing it. So Entity Sentinel gives you up to five tiers, each with its own targets and its own freeze window, all inside one sensor. Fast devices are held to a fast standard, slow ones to a slow one, and one evaluation judges every entity against its own tier's clock. Most homes need only three tiers; unused slots simply never appear.

**What it flags.** Each watched entity is checked for five conditions: `unavailable` (the state is unavailable), `unknown` (the state is unknown), `missing` (the entity does not exist: renamed, removed, or never created), `frozen` (the entity holds a value but the device has not reported within its tier's window), and `never_reported` (the entity exists but has never produced a reading).

**Two levels of judgment.** `unavailable`, `unknown`, `missing`, and `never_reported` are read straight from the entity you named. `frozen` is judged, and **reported**, at the **device** level: the freshest report from any entity on the same device counts, so a quiet entity whose sibling is still chatting is never flagged, the device is demonstrably alive. And when a device truly dies, it appears on the list once, represented by its highest-priority watched entity, no matter how many of its entities you watch, because one dead device is one problem, not three. A motion sensor is the classic case: its motion entity may be silent for hours in an empty room while its signal strength updates constantly; the device is fine, and Entity Sentinel knows it.

**The freeze clock.** Which clock supplies that device freshness matters more than anything else on this page. Where the device exposes a **last-seen entity** (a sibling with `device_class: timestamp` and `last_seen` in its entity id, as Zigbee2MQTT and Z-Wave JS publish), Entity Sentinel reads that entity's value: the protocol's own record of the last real message, which survives Home Assistant restarts and integration reconnects untouched. Where no such entity exists, it falls back to Home Assistant's `last_reported` metadata, which is less reliable, because restarts and re-publishes reset it. The practical consequence, and how to set yourself up well, is covered in its own section below.

## Before You Build: Two Small Prerequisites

1. **An uptime sensor in timestamp mode.** Settings, Devices & Services, Add Integration, search for Uptime, add it. Entity Sentinel uses it to know when Home Assistant started, which drives the startup grace that keeps reboots from causing false alarms. The default entity is `sensor.uptime`.
2. **A refresh button (optional but recommended).** Settings, Devices & Services, Helpers, Create Helper, Button. Name it something like Entity Sentinel. Pressing it forces an immediate re-evaluation, useful while you are setting up and any time you want an answer now instead of at the next scan. One button can be shared across every Sentinel you run.

## Import the Blueprint

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Ftemplate%2Fentity_sentinel.yaml)

Or paste this URL into Settings, Automations & Scenes, Blueprints, Import Blueprint:

```
https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/template/entity_sentinel.yaml
```

Your imported copy is a snapshot; re-import to pick up newer releases, and check the version in the blueprint description.

## Build the Sensor

This is a **template** blueprint, so importing only registers it; there is no _Create_ button and it will not appear in your automations list. It becomes a sensor through a `use_blueprint` entry, which you can add in one of two places.

**The quick way, straight in `configuration.yaml`.** Good for trying it out or if this is the only Sentinel you plan to run. Add the block below under a `template:` key in `configuration.yaml` (if you already have a `template:` key, add this as another entry under it rather than making a second one), then restart.

**The organized way, a package file.** Better once you run more than one Sentinel, or if you like keeping related configuration together. A [package](https://www.home-assistant.io/docs/configuration/packages/) is a file whose contents fold into your configuration at startup. Enable packages once in `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Then make a `packages` folder inside your `config` folder, create a file in it, for example `entity_sentinel.yaml` (the `.yaml` extension is required; a file without it is silently ignored), and paste the block below into that file. A brand-new package file registers at startup, so restart once; after that, edits to the same file reload through Developer Tools, YAML, Reload Template Entities.

Here is the starting block, with what every line is for:

```yaml
template:
  - use_blueprint:
      path: TheThinkingHome/entity_sentinel.yaml
      input:
        sensor_name: Entity Sentinel
        unique_id: entity_sentinel
        debug_enabled: false
        scan_interval: "/30"
        unavailable_debounce:
          minutes: 3
        startup_grace_seconds: 240
        uptime_sensor: sensor.uptime
        refresh_button: input_button.entity_sentinel
        tier_1_name: Lively
        tier_1_target:
          label_id:
            - entity_watch_lively
        tier_1_freeze:
          hours: 6
        tier_2_name: Quiet
        tier_2_target:
          label_id:
            - entity_watch_quiet
        tier_2_freeze:
          hours: 12
        tier_3_name: Sleepy
        tier_3_target:
          label_id:
            - entity_watch_sleepy
            - battery_watch
        tier_3_freeze:
          hours: 26
```

Line by line:

- **`path`** points at the imported blueprint. If you imported using the link above, it is exactly `TheThinkingHome/entity_sentinel.yaml`; the path is read relative to `config/blueprints/template/`.
- **`sensor_name`** is the display name of the sensor this block creates.
- **`unique_id`** registers the sensor so it is editable in the interface. Any unique string; never change it afterward, or Home Assistant treats it as a new entity.
- **`debug_enabled`** writes one summary line per evaluation to the log when `true`. Leave `false` for everyday running; see Parameters for how to surface the lines.
- **`scan_interval`** is how often the sensor re-evaluates, in minutes. `"/30"` is every thirty minutes, plenty for freeze windows measured in hours.
- **`unavailable_debounce`** is how long an entity must stay unavailable or unknown before it is flagged, so a two-second Zigbee blip never alerts.
- **`startup_grace_seconds`** holds the unavailable list empty for a few minutes after Home Assistant starts, while integrations and the mesh come up.
- **`uptime_sensor`** is the timestamp sensor from the prerequisites.
- **`refresh_button`** is the optional button from the prerequisites.
- **`tier_N_name` / `tier_N_target` / `tier_N_freeze`** define each tier: its display name, what it watches (the labels you will create next), and how long silence is allowed to last there. Tiers 4 and 5 are simply absent here; unused slots never appear.

The windows above, 6, 12, and 26 hours, are **starting points, not answers**. Your hardware, your network, your coordinator, and how your devices actually behave matter more than anyone can predict from outside your house. Expect to tune: a tier that false-alarms wants a longer window or a heartbeat entity (below); a tier that catches too slowly wants a shorter one. Earlier versions shipped longer suggestions because false triggers were harder to avoid; the current freeze clock (the last-seen mechanism below) has mitigated most of that, so tighter windows are now a reasonable place to begin.

## Verify It Is Alive

Before investing in labels, confirm the plumbing. Restart (first time) or reload template entities (later edits), then open Developer Tools, States, and search for your `sensor_name`. A healthy, not-yet-configured sensor looks like this: state `0`, `ok: true`, and a `tiers` attribute whose entries carry a `status` note saying the target resolved to nothing, which is exactly right, since nothing is tagged yet. If the sensor does not appear at all, the troubleshooting section below has the six causes in order of likelihood.

## Label Your Entities

Labels are how you tell each tier what to watch. Settings, Areas, labels & zones, Labels tab, create one label per tier: `entity_watch_lively`, `entity_watch_quiet`, `entity_watch_sleepy`. Then walk your critical entities: open each entity's settings (or use the entities list, select several, and apply a label in bulk) and tag it into the tier that matches how often it normally speaks. Chatty things like temperature and lux sensors go to Lively; event-driven things like motion sensors and regularly used contacts go to Quiet; devices that check in rarely, leak sensors, seldom-opened doors, go to Sleepy.

Two notes worth knowing. A tier's target takes a list, so one tier can watch several labels on one window, the sample's Sleepy tier watches `battery_watch` too, which catches battery entities gone silent (a different question from Battery Sentinel's low-charge detection; run both). And adding a device to the watch later is just applying the label to it, no package edit, no restart.

## Enable last_seen, the Most Reliable Freeze Detector

For every device you watch for freezes, enable its **last-seen entity** if it has one. This is the single highest-value setup step on this page.

The last-seen entity is the protocol's own record of the device's most recent real message. Zigbee2MQTT and Z-Wave JS both publish one per device (usually disabled by default): open the device's page, show its disabled diagnostic entities, and enable the one named last seen. For Zigbee2MQTT, also confirm the bridge setting that publishes it (`last_seen` under advanced settings, set to an ISO format) if the entity does not exist at all. The entity needs no label and no tier; it only needs to be enabled, and Entity Sentinel finds it by itself through the device.

Why it matters: Entity Sentinel reads that entity's **value** as the device's last report, and that value survives Home Assistant restarts and integration re-publishes untouched. It is the truth, kept by the protocol, immune to the resets described next.

## Without last_seen: the Fallback, and Its Failure

Where a device has no last-seen entity, Entity Sentinel falls back to Home Assistant's `last_reported` metadata, and you must understand how that clock fails: **`last_reported` is reset by any event that re-pushes states.** Each reset rewinds the freeze clock to that moment, silently un-flagging a standing freeze and restarting detection from zero. Events that reset it:

1. **A Home Assistant restart.** Every entity in the system is re-pushed at startup, so every fallback freeze clock resets at every restart, scheduled or not.
2. **An integration reload.** Reloading an integration (or a template reload that recreates entities) re-pushes its entities' states.
3. **A connection loss and recovery.** An integration that reconnects and republishes, Zigbee2MQTT after losing a network-attached coordinator, an MQTT broker restart, a cloud integration re-syncing, stamps every one of its entities fresh.
4. **A network event that triggers any of the above.** A router reboot that briefly severs the path to a network-attached coordinator resets every clock behind it, even though Home Assistant itself never restarted.

The consequence, stated plainly: **on the fallback path, a freeze window longer than the time between resets can never fire.** A system that restarts nightly can never detect a fallback freeze with a window longer than a day; the clock is rewound before it matures. A frozen device is still caught eventually, detection restarts after each reset, and the `unavailable` path, which no republish erases, remains the backstop. But the fallback is the less reliable clock, and the last-seen entity is the fix.

For devices that will never have a last-seen entity, a Wi-Fi presence sensor, for example, give the device a genuinely fast-reporting sibling to vouch for it (a light or signal-strength entity that updates constantly), size its freeze window shorter than your restart cadence, and let the unavailable detection cover the rest. Groups and other helpers deserve a special mention: they have no device behind them at all, no siblings, no last_seen, so their own state changes are their only signal, and a group that legitimately sits still for a day will read as frozen on a shorter window. Watch the members instead of the group where you can.

## Parameters

The five tier slots take the same four inputs each; N is 1 through 5.

| Parameter | Required | Default | What it does |
| --- | --- | --- | --- |
| `tier_N_name` | No | `Tier N` | The display name for the tier, carried on every flagged entry (`tier`) and in the per-tier counts. |
| `tier_N_target` | No | empty | Entities, labels, areas, or devices for this tier. Explicit entities and labels are watched precisely; areas and devices are swept to one representative entity per device. Empty means the tier is unused. |
| `tier_N_freeze` | No | 8h / 16h / 36h / 36h / 36h | How long a device in this tier may go without reporting before it counts as frozen, judged against the freshest report across the whole device (the last-seen value where one exists). |
| `tier_N_exclude` | No | empty | Entities, labels, areas, or devices to leave out of **this tier only**. The tool for resolving a cross-tier duplicate without touching the label. |
| `exclude_target` | No | empty | Entities, labels, areas, or devices to leave out of **every** tier. Exclude always wins over any include. |
| `unavailable_debounce` | No | 3 minutes | How long an entity must stay unavailable or unknown before it is flagged, whatever its tier. |
| `scan_interval` | No | `/2` | How often the sensor re-evaluates, in minutes. `/2` is fine for testing; raise it for everyday running: `/30` is every thirty minutes, or `"0"` is once an hour on the hour. Echoed as an attribute, so dashboards can display the cadence. |
| `startup_grace_seconds` | No | `240` | How long after Home Assistant starts to hold the unavailable list empty. Freeze is not held (it cannot false-fire at startup). |
| `uptime_sensor` | Yes | `sensor.uptime` | The uptime sensor in TIMESTAMP mode. Missing or non-timestamp is a `setup_error`. |
| `refresh_button` | No | none | An optional `input_button` for an immediate re-evaluation. Share one button across Sentinels. |
| `sensor_name` | No | `Entity Sentinel` | The name of the created sensor. |
| `unique_id` | Yes | none | A unique id so the sensor is registered and editable in the interface. |
| `debug_enabled` | No | `false` | One summary log line per evaluation, with per-tier counts and every flagged entity. To see the lines, add the logger at info level in `configuration.yaml`: under `logger:`, `logs:`, add `entity_sentinel: info`. |

## Attributes

Attributes are ordered so the useful summary sits on top and the reference fields below: `ok`, `error`, `total_monitored`, `unavailable_count`, `frozen_count`, `tier_error_count`, `devices` (each entry: `name`, `entity_id`, `area`, `tier`, `reason`, `since`, `last_seen`, `age`), `tiers` (per active tier: `tier`, `monitored`, `flagged`, `status`), then `sentinel_type`, `sentinel_version`, `scan_interval`, `uptime_status`, `boot_time`, `settled`, `last_evaluated`. The state is the merged flagged count. The list is ordered by area, then name.

A sensor with one frozen entity looks like this:

```yaml
state: 1
ok: true
error: ""
total_monitored: 42
unavailable_count: 0
frozen_count: 1
tier_error_count: 0
devices:
  - name: Laundry Motion
    entity_id: binary_sensor.laundry_motion_occupancy
    area: Laundry
    tier: Quiet
    reason: frozen
    since: "2026-07-09T13:02:11+00:00"
    last_seen: "2026-07-09T13:02:11+00:00"
    age: 12 hours
tiers:
  - tier: Lively
    monitored: 12
    flagged: 0
    status: ""
  - tier: Quiet
    monitored: 9
    flagged: 1
    status: ""
  - tier: Sleepy
    monitored: 21
    flagged: 0
    status: ""
sentinel_type: entity
sentinel_version: 2.2.0-beta
scan_interval: /30
uptime_status: "2026-07-09T08:42:08+00:00"
boot_time: "2026-07-09T08:42:08+00:00"
settled: true
last_evaluated: "2026-07-10T01:30:00.421000-05:00"
```

Three tier rules worth knowing while you read that: a tier with no targets is skipped silently (most homes need only three, so empty slots are normal; all five empty is a loud `setup_error`); an entity may live in exactly one tier, priority runs 1 over 2 over 3 over 4 over 5, a duplicate stays monitored in its lowest-numbered tier, and any higher-numbered tier that also claims it drops it and names it in that tier's `status` so you can remove it or exclude it with `tier_N_exclude`; and a tier watching nothing is a note, not an error, its `status` tells you why, so a half-finished setup never looks broken.

## Decisions and Limitations

- **One sensor, five fixed tier slots.** Blueprint inputs cannot be dynamic, so the slots are hardcoded; empty ones never appear. Tiers replaced the old pattern of one sensor per freeze window, one block now does what several sensors and an aggregator once did.
- **Duplicates are reported, never silently resolved.** An entity claimed by two tiers is an error you are told about, in the higher tier's `status` and in `tier_error_count`, while everything healthy keeps rendering. The sensor stays `ok: true`; a tier-level mistake never blinds the whole sensor.
- **Freeze is device-level; everything else is entity-level.** A chatty sibling vouches for a quiet one. The flip side: excluding a swept device's elected representative drops that device from an area sweep entirely.
- **A Home Assistant restart wipes `last_reported`, for everything.** So does any integration re-publish. On the fallback clock this rewinds freeze detection to zero; on the last-seen clock it does not. This is the single most important limitation to understand, and the reason the last-seen section above exists.
- **A freeze window longer than your reset cadence cannot fire on the fallback path.** Size fallback windows shorter than the time between your restarts, or better, eliminate the fallback with last-seen entities.
- **Groups and helpers have no device.** No siblings, no last-seen; their own silence is their only signal. Prefer watching members.
- **Freeze is deliberately exempt from the startup grace.** It cannot false-fire at a restart (a restart makes the clock younger, never older), so it is not held.
- **Availability systems are the backstop, not the rival.** Zigbee2MQTT marks a passive device unavailable after roughly 25 hours; Entity Sentinel's unavailable detection picks that up regardless of any freeze clock. The two paths cover each other.

## Upgrading from 1.x

The 1.x line needed one sensor per freeze window plus an aggregator to merge them; 2.x replaced all of that with tiers in one sensor. If you run 1.x: re-import (the blueprint updates in place, and its inputs are different, so migrate in the same sitting), fold each old copy into a tier of one new block, its `include_target` becoming `tier_N_target` and its freeze window `tier_N_freeze`, restart, and repoint anything that read the old sensors. Your labels carry over untouched. The frozen 1.x line stays downloadable as [entity_sentinel_1.0.5-beta.yaml](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/entity_sentinel_1.0.5-beta.yaml) and [entity_sentinel_1.0.5-beta.md](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/entity_sentinel_1.0.5-beta.md).

## If the Sensor Does Not Appear

In order of likelihood:

1. **The package file is not named `*.yaml`.** A missing extension is silently ignored.
2. **Packages are not enabled.** Check the `homeassistant: packages:` include in `configuration.yaml`.
3. **The `path:` is wrong.** It must be exactly `TheThinkingHome/entity_sentinel.yaml`.
4. **No restart since the file was created.** A brand-new package file registers only at startup; reload covers edits to an existing file, not new files.
5. **The blueprint is not imported.** Check Settings, Automations & Scenes, Blueprints for Entity Sentinel.
6. **A YAML error in the package.** Run the configuration check (Developer Tools, YAML, Check Configuration) and read the error it names.

If the sensor exists but reports `setup_error`, read its `error` attribute: it names the fault (an uptime sensor that is missing or not in timestamp mode, or no tiers configured).

## Changelog

| Version | Notes |
| --- | --- |
| 2.2.0-beta | Frozen reports per device, and area-then-name order. One dead device carries several watched entities that all freeze together, so the list showed the same dead device once per entity. Frozen entries now collapse to one per device, represented by the entity from its highest-priority tier; `frozen_count` counts frozen devices. Deviceless entities still stand alone; `unavailable`, `unknown`, `missing`, and `never_reported` remain per-entity. The list is now ordered by area, then name. |
| 2.1.1-beta | Exposes the `scan_interval` input as an attribute, so dashboards can display the evaluation cadence. No behavior change. |
| 2.1.0-beta | The device-truth freeze clock. Freeze is now judged, where available, from a device's own last-seen entity value (a sibling with `device_class: timestamp` and `last_seen` in its entity id, as Zigbee2MQTT and Z-Wave JS publish), falling back to `last_reported` where no such entity exists. Why: restarts and integration re-publishes reset `last_reported` for every entity, silently rewinding the freeze clock; on a system with a nightly network event, a freeze window longer than the reset cadence could never fire. The last-seen value carries the protocol's truth through every republish. Found live: two physically dead planted devices were swept as recovered by a router-reboot republish. No input or output change; proven by a six-check republish wave in the 63-check suite. |
| 2.0.0-beta | The graduation release. Banked behind it: a 57-check adversarial simulation suite, two external code reviews with every finding reproduced against the file before acting (one real counting bug found and fixed), and a week of live duty through nightly reboot cycles and real detections. The engine unchanged from 2.0.0-alpha.1.2; the inputs and the output contract are settled from this release on. |
| 2.0.0-alpha.1.2 | Count bucketing fix. An entity whose state is the literal string `none` was flagged in `devices` and counted in the state, but fell into neither `unavailable_count` nor `frozen_count`. `none` now buckets into `unavailable_count`. Found by an externally proposed test case. |
| 2.0.0-alpha.1.1 | Attribute presentation only; no logic change. `tiers` entries show `tier: <name>`; `error` and `note` merged into one `status` string; attributes reordered summary-first. |
| 2.0.0-alpha.1 | The tiered rebuild. Up to five tiers in one sensor, each with its own targets and freeze window on one shared cadence and one shared unavailable debounce. Output contract a strict superset of 1.x. |

The 1.x line and its full history live in the frozen [entity_sentinel_1.0.5-beta.md](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/entity_sentinel_1.0.5-beta.md).

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>). This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.
