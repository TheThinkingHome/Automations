# Entity Sentinel (2.0 Development)

One sensor, up to five tiers. Each tier carries its own targets and its own freeze window; the sensor evaluates every tier on one shared cadence and publishes one merged, tier-annotated list. This replaces the pattern of building one Entity Sentinel per cadence class plus [Dashboard Sentinel](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/dashboard_sentinel.md) to combine them: the tiers live inside the detector.

**This is the development line, currently 2.0.0-alpha.1.** It has passed a first-light bench, not yet the full torture gate, and it is not yet running anywhere. For a settled setup today, use the stable [Entity Sentinel](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/entity_sentinel.md) and, to combine several of them, [Dashboard Sentinel](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/dashboard_sentinel.md). Everything those two do together, this one sensor will do alone.

The output contract is a strict superset of Entity Sentinel 1.x: the same `devices` shape and attributes, so [Sentinel Notify](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/automation/sentinel_notify.md) and your dashboard cards consume it unchanged. New: each `devices` entry carries `tier`, a `tiers` attribute reports per-tier counts and per-tier errors, and `tier_error_count` is the badge hook.

## How the tiers work

Give each tier the entities that share a reporting rhythm and a freeze window matching that rhythm. One evaluation on one cadence judges every entity against its own tier's window, so a slow device can never false-flag on a fast window. Unavailable and unknown entities are flagged after one shared debounce, whatever their tier.

Three rules:

1. **A tier with no targets is skipped silently.** Unused slots never appear in your package. All five empty is a loud `setup_error`.
2. **An entity may live in exactly one tier.** Tier priority is 1 over 2 over 3 over 4 over 5: a duplicate stays monitored in its lowest-numbered tier, and every higher-numbered tier that also contains it excludes it and names it in that tier's `error`, while the tier's healthy entities keep rendering. The sensor stays `ok: true`; `tier_error_count` tells you a tier needs fixing.
3. **A target that resolves to no entities is a `note`, not an error.** A freshly created label with nothing tagged yet is a normal mid-setup state.

## Import the blueprint

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Ftemplate%2Fentity_sentinel_2.yaml)

Or paste this URL into Settings, Automations & Scenes, Blueprints, Import Blueprint:

```
https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/template/entity_sentinel_2.yaml
```

This is a template blueprint, so importing only registers it; there is no Create button. It becomes a sensor through a `use_blueprint` entry in a package file. The [Entity Sentinel README](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/entity_sentinel.md) walks the package setup step by step (the folder, the **`.yaml` filename requirement**, the path rule), and the [packages documentation](https://www.home-assistant.io/docs/configuration/packages/) covers packages in full. The `path:` is relative to `config/blueprints/template/`, so only the last part is used, exactly `TheThinkingHome/entity_sentinel_2.yaml`.

## A sample configuration

Four tiers on a real system: three cadence classes plus a battery-liveness tier (which detects battery *entities gone silent*, a different question from Battery Sentinel's low-level detection; run both). Tier 5 is simply absent, unused slots never appear:

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
        tier_3_freeze:
          hours: 36
        tier_4_name: Batteries
        tier_4_target:
          label_id:
            - battery_watch
        tier_4_freeze:
          hours: 36
```

Restart once (a brand-new package file registers at startup); after that, edits to the same file reload through Developer Tools, YAML, Reload Template Entities.

## Parameters

The five tier slots take the same three inputs each; N is 1 through 5.

| Parameter | Required | Default | What it does |
| --- | --- | --- | --- |
| `tier_N_name` | No | `Tier N` | The display name for the tier, carried on every flagged entry (`tier`) and in the per-tier counts. |
| `tier_N_target` | No | empty | Entities, labels, areas, or devices for this tier. Explicit entities and labels are watched precisely; areas and devices are swept to one representative entity per device. Empty means the tier is unused. |
| `tier_N_freeze` | No | 8h / 16h / 36h / 36h / 36h | How long a device in this tier may go without reporting anything before it counts as frozen, judged against the freshest report across the whole device. |
| `exclude_target` | No | empty | Entities, labels, areas, or devices to leave out of **every** tier. Exclude always wins over any tier's include. |
| `unavailable_debounce` | No | 3 minutes | How long an entity must stay unavailable or unknown before it is flagged, whatever its tier. |
| `scan_interval` | No | `/2` | How often the sensor re-evaluates, in minutes. The only latency knob: an entity is caught within one tick of its own tier's window expiring. |
| `startup_grace_seconds` | No | `240` | How long after Home Assistant starts to hold the unavailable list empty while the system and mesh come up. Freeze is not held (it cannot false-fire at startup). |
| `uptime_sensor` | Yes | `sensor.uptime` | The uptime sensor in TIMESTAMP mode. Missing or non-timestamp is a `setup_error`. |
| `refresh_button` | No | none | An optional `input_button` for an immediate re-evaluation. Share one button across Sentinels. |
| `sensor_name` | No | `Entity Sentinel` | The name of the created sensor. |
| `unique_id` | Yes | none | A unique id so the sensor is registered and editable in the interface. |
| `debug_enabled` | No | `false` | One summary log line per evaluation, with per-tier counts and every flagged entity. |

## Attributes

Everything Entity Sentinel 1.x publishes, unchanged: `ok`, `error`, `sentinel_type`, `sentinel_version`, `uptime_status`, `total_monitored`, `devices` (each entry: `name`, `entity_id`, `area`, `reason`, `since`, `last_seen`, `age`), `unavailable_count`, `frozen_count`, `boot_time`, `settled`, `last_evaluated`. The state is the merged flagged count, entity-id sorted.

New in 2.0:

| Attribute | What it carries |
| --- | --- |
| `tier` (on each `devices` entry) | The display name of the tier that flagged the entity. |
| `tiers` | One entry per active tier: `tier` (slot number), `name`, `monitored`, `flagged`, `error` (duplicates, naming each entity), `note` (zero-resolving target). |
| `tier_error_count` | How many tiers currently report an error. Zero when the configuration is clean. |

## If the sensor does not appear

The checklist in the [Entity Sentinel README](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/entity_sentinel.md) applies verbatim, with one substitution: the `path:` is `TheThinkingHome/entity_sentinel_2.yaml`.

## Changelog

| Version | Notes |
| --- | --- |
| 2.0.0-alpha.1.1 | Attribute presentation only; no logic change. The engine does exactly what alpha.1 did. Three improvements to how the `tiers` attribute reads and how attributes are ordered: each `tiers` entry now shows `tier: <name>` instead of a `tier: <number>` plus a separate `name:` field; the old `error` and `note` fields are merged into a single `status` string (empty when the tier is healthy), which also now distinguishes a tier whose own exclude removed every resolved entity from one whose target simply resolved to nothing; and the attributes are reordered so the useful summary sits on top (`ok`, `error`, `total_monitored`, the counts, `devices`, `tiers`) with the reference fields below. `tier_error_count` is unchanged in meaning. |
| 2.0.0-alpha.1 | The tiered rebuild. Up to five tiers in one sensor, each with its own targets and freeze window on one shared cadence and one shared unavailable debounce. Duplicates across tiers resolve to the lowest-numbered tier and are reported as the higher tier's error, with partial rendering. Output contract is a strict superset of 1.x: `tier` on each entry, the `tiers` attribute, `tier_error_count`, entity-id sorted list. Supersedes the multi-sensor pattern and the Dashboard Sentinel aggregator for new builds. First-light bench, 12 checks. |

The 1.x line and its full history live in the [Entity Sentinel README](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/entity_sentinel.md).

## License

GPL-3.0-or-later. Copyright (C) 2026 James Lander, The Thinking Home ([xeazy.com](https://xeazy.com)).
