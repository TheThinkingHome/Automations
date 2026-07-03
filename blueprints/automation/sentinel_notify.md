# Sentinel Notify

> **This is the active development branch.** The blueprint here is being rebuilt toward 2.0.0 and is not yet stable, not yet fully hardware-tested, and its inputs and behavior will change between commits. **If you want Sentinel Notify running in your home today, use the stable branch below.**

## The Stable Release: 1.0.0-alpha.7.2

The current proven release is **1.0.0-alpha.7.2**. It runs live, has survived an external code review and an adversarial simulation suite, and its behavior is frozen: no further changes except critical fixes. It works differently from what is being built here, so read its own documentation:

- [**Stable README**](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/automation/sentinel_notify_1.0.0a7.2.md), setup, parameters, and behavior.
- [**Stable blueprint YAML**](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/automation/sentinel_notify_1.0.0a7.2.yaml), the file to import.

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Fautomation%2Fsentinel_notify_1.0.0a7.2.yaml)

Or paste this URL into Settings, Automations & Scenes, Blueprints, Import Blueprint:

```
https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/automation/sentinel_notify_1.0.0a7.2.yaml
```

If you imported Sentinel Notify from this repository before, your imported copy is a snapshot and keeps working as-is. For any future re-import, use the stable URL above rather than the main `sentinel_notify.yaml`, which now carries development builds.

## What Sentinel Notify Is

Sentinel Notify is the notification companion to the two Sentinel sensor blueprints. It does nothing on its own: build a [**Battery Sentinel**](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/battery_sentinel.md) or an [**Entity Sentinel**](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/entity_sentinel.md) first (each makes a sensor), then build this to notify from it. It reads the rich attributes a Sentinel exposes and tells you what is actually wrong, by name, with the detail, and only when the situation changes, so it does not flood you every cycle. The full design and worked examples are in the [article](https://xeazy.com/battery-entity-sentinel-blueprints/).

## What 2.0 Is About

The 1.x engine remembers what it last reported as a hashed fingerprint inside an `input_text` helper. The hash is compact and reliable, but it can only answer one question: did the set change? It cannot say what changed, and the helper's 255-character limit is why a hash is used at all.

2.0 replaces that memory with a **Local To-do list**. Each flagged item becomes one entry on the list, which changes what the companion can do:

- **Unlimited memory, no hash.** The list holds the actual items, however many there are. The engine can do real set arithmetic on them.
- **Notifications name what is new.** Instead of re-listing everything currently flagged, an alert can say which entity or battery was just added. An item leaving the list (a recovery) is handled without re-announcing the ones that remain, which also finishes off a whole class of false alerts around restarts and staggered device recovery.
- **The list is the report.** A Battery Sentinel list reads as a shopping list, name, battery type, and area per line, visible in a dashboard to-do card and in the companion app. An Entity Sentinel list is a live outage board. The list state is the count of open items, so a badge or conditional card can key on it directly.
- **The checkbox means something.** Checking off an item acknowledges it: it stays known to change detection (so it never re-notifies), it is left out of the daily summary, and it re-arms by itself only if the entity recovers and then fails again, a genuinely new incident. A recovered item is removed from the list automatically.
- **The `input_text` helper stays, smaller.** It keeps holding the broken-Sentinel state and the quiet-hours held flag; the item memory moves to the list.

The trade: setup gains one step (each copy needs its own Local To-do list, created through Add Integration rather than the Helpers page), and the engine restructure is large enough that 2.0 starts life as an alpha all over again, on the same deal as before: I test as hard as I can in one house and in simulation, and the community's use is what makes it solid.

## Import the Development Build

If you want to follow the 2.0 line as it develops, help test, and can tolerate breaking changes between commits, import from this branch. Everyone else should use the stable import above.

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Fautomation%2Fsentinel_notify.yaml)

Or paste this URL into Settings, Automations & Scenes, Blueprints, Import Blueprint:

```
https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/automation/sentinel_notify.yaml
```

Your imported copy is a snapshot; re-import to pick up newer development builds. Check the Status table below and the version number in the blueprint description before relying on it.

## Status

| Stage | State |
| --- | --- |
| Design (to-do memory, addition-only gate, acknowledgment semantics) | Agreed |
| First build (2.0.0-alpha.8) | In progress |
| Bench testing against the adversarial simulation suite | Pending |
| Live shadow deployment (engine runs, notifications muted, verified across nightly reboots) | Pending |
| Beta graduation | Pending |

Nothing on this branch is importable for daily use until this table says otherwise and the version drops its alpha tag. Progress, questions, and bug reports live in the [community thread](https://xeazy.com/logbook/d/42-the-battery-entity-sentinel-blueprints).

## Changelog

The 2.0 changelog starts when the first build lands here. The full 1.x history is in the [stable README](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/automation/sentinel_notify_1.0.0a7.2.md#changelog).

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>). This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.
