# Sentinel Notify (Alpha)

A Home Assistant automation blueprint that turns a Battery Sentinel or Entity Sentinel into smart, change-aware notifications: it tells you what is actually wrong, by name, and does not SPAM your notifications.

## What This Solves

The Sentinels produce a sensor with a rich set of attributes: not just a count, but a list of exactly which batteries are low (with percentage, battery type, and area) or which entities have gone quiet (with the reason and how long). That detail is the whole point of building a sensor instead of a one-shot notifier. But a sensor does not notify you on its own; you need something that reads it and decides when and how to alert you.

This is a companion to the two Sentinel sensor blueprints, and it does nothing on its own. Build a Sentinel first, it creates the sensor, then build this to notify from it:

- [**Battery Sentinel**](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/battery_sentinel.md) counts the batteries running low.
- [**Entity Sentinel**](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/entity_sentinel.md) catches entities that have gone quiet, unavailable or frozen at their last value.

Because it reads the structured attributes, it can do two things the common notifier blueprints cannot. It can name what is wrong, "Master Bath Motion (12%, CR2450)", not just "3 batteries low." And it can notify you only when the situation has changed, so it does not nag you every cycle about the same three batteries or entities you already know about. This companion comes from **The Thinking Home** at [xeazy.com](https://xeazy.com). The full design and worked examples are in the [article](https://xeazy.com/battery-entity-sentinel-blueprints/).

## Built on Request, and Why It Is a Beta

Every other blueprint from The Thinking Home grew out of my own house. The sensors and automations behind them ran on my system for years before I shared them, so the reliability was already settled. My job was to generalize what already worked: make it configurable, document it, smooth the edges, and hand over something I already trusted.

This one is different. It did not run quietly for years in my walls. It was asked for, by the community, for a problem the existing tools cannot solve. So I built it: imagined, drafted, edited, reimagined, debugged, and am still testing it as hard as I can in one house and an adversarial test suite allows.

That is why this is an **alpha** release. It is genuinely useful today, and I run it on my own system. But if you use it, you are also helping test and refine it. Real homes are more varied than any one setup, and the edge cases that matter most are the ones I did not imagine and cannot produce. If you hit something, the [community thread](https://xeazy.com/logbook/d/42-the-battery-entity-sentinel-blueprints) is where it gets found and fixed. That is the deal: I have done my best to build it, and the community's use is what will make it solid.

## How it Works

This companion watches one Sentinel family at a time. You set a mode, _Battery or Entity_, and point it at sensors from that family. If you run both Sentinels, build two copies of this companion. They behave differently enough, batteries are sparse and slow, entities are denser and twitchier, that keeping them separate makes each one simpler to configure and its notifications cleaner to read.

## What It Does

- **Reads the Sentinel's attributes** and reports the actual devices: for batteries, the name, percentage, battery type, and area; a binary low-battery sensor (which has no percentage) is reported as "(low)".
- **Notifies only on change.** It checks on a schedule you set, but sends a notification only when the set of flagged devices is different from what it last told you about. The same unchanged situation stays quiet.
- **Combines several sensors into one report.** Point it at all your Battery Sentinel sensors and a single notification covers them all.
- **Sends where you want.** Push to any number of mobile devices, and optionally a persistent notification in the Home Assistant UI, replaced in place so it never stacks.
- **Respects priority.** Normal, high (bypasses Do Not Disturb, alarm sound on Android, time-sensitive on iOS), or low (silent).
- **Optional re-reminder.** Off by default. If you want a nag about batteries you have not replaced, set a re-remind interval and it re-sends the current report on that schedule even when nothing has changed.
- **Fails loud if set up wrong.** A wrong-family sensor, a missing memory helper, or the unbuilt Entity mode each stop with a clear log message rather than failing quietly.

The full design and worked examples are in the article: <https://xeazy.com/battery-entity-sentinel-blueprints/>

## Before You Start: the Memory Helper

This companion needs one helper to do its job: an `input_text` it uses to remember what it last reported, so it can notify you only on a change.

Create it first: Settings, Devices & Services, Helpers, Create Helper, Text. Give it any name, for example "Battery Notify Memory." You will select it in the blueprint below.

> **Critical: one dedicated helper per copy.** Every copy of this companion must have its own input_text helper. Two copies (say a Battery one and an Entity one) sharing a single helper will overwrite each other's memory and cross-fire endless notifications, each thinking the situation changed on every other check. If you build more than one copy, create a separate helper for each.

You do not need to set a maximum length or any options on the helper; the default is fine. The companion stores a compact fingerprint there, which fits comfortably regardless of how many devices you watch.

## Import the Blueprint

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Fautomation%2Fsentinel_notify.yaml)

Or paste this URL into Settings, Automations & Scenes, Blueprints, Import Blueprint:

```
https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/automation/sentinel_notify.yaml
```

## Setting Up the Automation

Unlike the Sentinels (which are template blueprints), this is an **automation** blueprint, so it sets up the easy way. After importing, go to Settings, Automations & Scenes, click Create Automation, and choose Sentinel Notify from the blueprint list. Fill in the form and save.

The form is organized into sections:

1. **Mode**, choose Battery (Entity arrives later).
2. **Sensors to Watch**, pick your Battery Sentinel sensor(s), and select the memory helper you created.
3. **Battery Settings**, optional toggles for the Battery mode.
4. **Entity Settings**, ignore for now (reserved for the Entity mode).
5. **Notification Targets and Style**, where and how to send.
6. **Advanced**, cadence, the optional re-reminder, and debug logging.

Because Home Assistant cannot hide form sections based on your mode choice, both the Battery and Entity sections are shown. Fill in the one matching your mode and leave the other alone.

## Parameters

| Input | Required | Default | What it does |
| --- | --- | --- | --- |
| Sentinel family | Yes | Battery | Battery or Entity. One family per copy. Entity arrives in a later beta. |
| Sentinel sensors | Yes | none | One or more Sentinel sensors of the chosen family. Combined into one report. |
| Memory helper | Yes | none | The `input_text` the companion uses to remember what it last reported. One per copy. |
| Also report offline batteries | No | off | Battery mode. When on, also reports batteries that are currently unavailable, alongside the low ones. |
| Push notification targets | No | none | The mobile devices to push to. Leave empty to use only the persistent notification. |
| Create a persistent notification | No | on | Also creates or updates a persistent notification in the HA UI, replaced in place. |
| Priority | No | normal | Push urgency. High bypasses Do Not Disturb; low is silent. |
| Notification tag | No | `sentinel_notify` | The tag that replaces this companion's push in place. Give each copy a distinct tag if several push to one device. |
| Check interval | No | `/5` | How often to re-check, in minutes. It notifies only on change, so a short interval keeps detection prompt without spamming. |
| Re-remind interval (hours) | No | 0 (off) | If above 0, re-sends the current report this often even when nothing changed. A nag for unresolved items. |
| Debug logging | No | off | Writes one diagnostic line per check. Needs a `logger` entry at info level to appear. |

## How "Notify Only on Change" Works

The companion keeps a small fingerprint of what it last reported, in the memory helper. Each check, it builds the same kind of fingerprint from what the Sentinel is flagging right now, sorted so the order never matters, and compares the two. If they differ, the situation has changed: it sends a notification listing everything currently flagged, and stores the new fingerprint. If they match, it stays silent.

So a battery dropping below the threshold, or one being replaced, changes the fingerprint and you hear about it. The same three batteries sitting low for a week do not re-notify you every cycle, you were told once. If you do want the periodic nag for unresolved items, the re-remind interval turns it on.

By design, the change is tracked at the level of which batteries are low, not their exact percentage. A battery that keeps draining while staying below the threshold (say 15% down to 5%) will not send a fresh alert, you were already told it is low. This is deliberate: tracking the exact percentage would re-notify on every reading and bring back the flood the change detection exists to prevent. If you want to be nudged about a battery you have not yet replaced, use the re-remind interval.

Because the notification uses a tag, even when it does fire it replaces the previous one in place rather than stacking a pile of alerts.

If one of the source Sentinels is itself broken (its uptime sensor was deleted, or it has gone unavailable), the companion sends a separate "Sentinel not working" notification, on its own tag, so it does not interfere with the battery report. A broken monitor never silences the healthy ones: low batteries on the working Sentinels are still reported in the same run.

## Keeping It Calm

The companion is only as quiet as the sensor you point it at. For batteries this is rarely an issue, batteries cross the threshold one at a time, so notifications are naturally infrequent. The advice matters more for the Entity mode (coming later): watch the handful of things whose silence is a real problem, not every entity in the house, because a sensor flagging dozens of flapping entities will notify you about churn no matter how good the change detection is. A tight, deliberate scope on the Sentinel is what keeps the notifications meaningful.

## Putting It to Work

The simplest setup: Battery mode, point it at your Battery Sentinel sensor, select a memory helper, add your phone as a push target, and leave the rest at defaults. You will get a notification when a battery drops below the threshold, named and with its battery type, and another when you replace it, with nothing in between.

For a quieter, dashboard-only setup, leave the push targets empty and keep the persistent notification on; the report lives in the Home Assistant UI and updates itself, with no phone alerts at all.

For the "remind me about what I keep ignoring" setup, set the re-remind interval to 24 hours, and a still-unreplaced battery nudges you once a day until you deal with it.

More worked examples are in the article: <https://xeazy.com/battery-entity-sentinel-blueprints/>

## Changelog

| Version | Notes |
| --- | --- |
| 1.0.0-beta | Initial release. Battery mode: reads Battery Sentinel sensors and reports low batteries (and optionally offline ones) by name with percentage, battery type, and area; binary low-battery sensors are reported without a percentage. Notifies only when the flagged set changes, using a sorted, hashed fingerprint stored in one `input_text` helper, so it does not notify every cycle. A device reported by two overlapping-scope sensors is de-duplicated to one entry. A separate, change-detected "Sentinel not working" alert fires when a source Sentinel is in a setup error or has gone unavailable, without suppressing the battery alerts from the healthy sensors. Push to any number of mobile devices and/or a persistent notification, tag-replaced in place, with priority handling for iOS and Android. Optional re-remind interval, off by default. The Entity mode is reserved in the same blueprint for a later release. |

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>). This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.

You will find the full YAML, this README, and the one-click import badge at the [Thinking Home blueprints repository](https://github.com/TheThinkingHome/Automations).
