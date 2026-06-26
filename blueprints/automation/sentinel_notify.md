# Sentinel Notify (Beta)

A Home Assistant automation blueprint that turns a Battery Sentinel or Entity Sentinel into smart, change-aware notifications: it tells you what is actually wrong, by name, and only when the situation changes.

## What This Solves

The Sentinels produce a sensor with a rich set of attributes: not just a count, but a list of exactly which batteries are low (with percentage, battery type, and area) or which entities have gone quiet (with the reason and how long). That detail is the whole point of building a sensor instead of a one-shot notifier. But a sensor does not notify you on its own; you need something that reads it and decides when and how to tell you.

That is this companion. And because it reads the structured attributes, it can do two things the common notifier blueprints cannot. It can name what is wrong, "Master Bath Motion (12%, CR2450)", not just "3 batteries low." And it can notify you only when the situation has changed since it last told you, so it never nags you every cycle about the same three batteries you already know about.

This is a companion to the two Sentinel sensor blueprints, and it does nothing on its own. Build a Sentinel first, it creates the sensor, then build this to notify from it:

- [**Battery Sentinel**](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/battery_sentinel.md) counts the batteries running low.
- [**Entity Sentinel**](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/entity_sentinel.md) catches entities that have gone quiet, unavailable or frozen at their last value.

Both, and this companion, come from **The Thinking Home** at [xeazy.com](https://xeazy.com). The full design and worked examples are in the [article](https://xeazy.com/battery-entity-sentinel-blueprints/), and discussion is in the [community thread](https://xeazy.com/logbook/).

## Built on Request, and Why It Is a Beta

Every other blueprint from The Thinking Home grew out of my own house. The sensors and automations behind them ran on my system for years before I shared them, so the reliability was already settled. My job was to generalize what already worked: make it configurable, document it, smooth the edges, and hand over something I already trusted.

This one is different. It did not run quietly in my walls for years before release. It was asked for, by the community, for a problem the existing tools did not solve. So I built it: imagined, drafted, edited, debugged, and tested as hard as one house allows. I believe in it, but I cannot yet claim what years of uneventful running would let me claim.

That is why this is a beta, not a stable release. It is genuinely useful today, and I run it on my own system. But if you use it, you are also helping test and refine it. Real homes are more varied than any one setup, and the edge cases that matter most are the ones I cannot produce here. If you hit something, the [community thread](https://xeazy.com/logbook/) is where it gets found and fixed. That is the deal: I have done my best to build it, and the community's use is what will make it solid.

## One Family Per Copy

This companion watches one Sentinel family at a time. You set a mode, Battery or Entity, and point it at sensors of that family. If you run both Sentinels, build two copies of this companion, one in each mode. They behave differently enough, batteries are sparse and slow, entities are denser and twitchier, that keeping them separate makes each one simpler to configure and its notifications cleaner to read.

This beta ships the **Battery** mode. The **Entity** mode is built into the same blueprint and arrives in a later beta; selecting it today reports a setup error pointing you back to Battery.

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
| 1.0.8-beta | The broken-Sentinel alert is now its own notification with its own tag and no longer halts the run, so a low battery on a healthy Sentinel is still reported when another Sentinel is down. Change-detection memory is now two hashed slots in one helper, so no upstream error text can corrupt the stored value or cause a re-alert loop. The offline-wins de-duplication uses explicit fields rather than a dictionary merge. |
| 1.0.7-beta | When two source Sentinels with overlapping scopes report the same physical device in different states (low from one, offline from the other), the companion now collapses it to one entry instead of double-counting with contradictory lines. Dedupe is by device, and offline wins over low, resolved order-independently. |
| 1.0.6-beta | Two fixes in the upstream-broken handling. An upstream sensor that is unavailable or unknown (integration failed to load, or it was deleted) is now alerted as a broken monitor, rather than slipping past the checks and being read as all-clear. The multi-sensor error string is joined with a comma instead of "||", which had collided with the stored fingerprint delimiter and caused an endless re-alert loop when two sensors broke at once. |
| 1.0.5-beta | Two fixes from a second adversarial review. A broken upstream Sentinel is now surfaced rather than hidden: if a source sensor reports a setup error (for example its uptime sensor was deleted), the companion sends a distinct "Sentinel not working" alert and stops, instead of reading the empty device list as all-clear and dismissing the warning. The alert is change-detected, firing once on break and clearing on recovery. The 1.0.4 source-sensor trigger condition compared only the count, which missed a real change when one battery cleared and another dropped in the same window; it now compares the device list, so a same-count swap is caught while attribute churn is still ignored. |
| 1.0.4-beta | Two fixes from an adversarial review. The plain-versus-hash fingerprint gate dropped from 240 to 200 characters so the stored value (fingerprint plus a timestamp) always fits the 255-character `input_text` limit; previously a mid-length fingerprint could overflow, fail its write silently, and re-notify every cycle. The source-sensor trigger now ignores attribute-only changes, so the companion no longer re-evaluates on every upstream cadence tick. |
| 1.0.3-beta | Hardening from an adversarial simulation. The battery line renderer no longer crashes on a malformed device item: a missing, null, or non-numeric level renders as "(low)", and a numeric level is coerced and shown as a clean integer percent (12.7 reads as 13%, not 13.0%). Flagged items are de-duplicated by entity, so a battery reported by two source sensors is counted and listed once. A correct Battery Sentinel does not emit malformed items, so this is defensive hardening for a public companion. |
| 1.0.2-beta | Fixed a push guard that could silently swallow notifications after a restart. The 1.0.1 `has_value` guard rejected a target in the `unknown` state, but a mobile_app notify entity sits at `unknown` until its first send of the session, so a battery dropping low shortly after a reboot would skip the push. The guard now skips only a genuinely `unavailable` target and lets an idle one through. |
| 1.0.1-beta | Pre-deploy review fixes. The persistent notification card is dismissed by a standalone, ungated step whenever nothing is flagged, and created or updated only when something is, removing the old create-then-dismiss dance and any chance of a ghost card. Push priority and ttl now scale with the chosen priority level, so a low-priority alert no longer wakes an Android radio at full power. Each push target is guarded with `has_value` so a renamed or deleted notify entity is skipped cleanly. The memory-helper guidance now stresses one dedicated helper per copy. |
| 1.0.0-beta | First beta. Battery mode: reads Battery Sentinel sensors, reports low batteries (and optionally offline ones) by name with percentage, battery type, and area; binary low-battery sensors reported without a percentage. Combines multiple sensors into one report. Notifies only when the flagged set changes, using a sorted fingerprint stored in an `input_text` helper, so it does not notify every cycle. Push to any number of mobile devices and/or a persistent notification, tag-replaced in place. Priority levels with iOS and Android handling. Optional re-remind interval, off by default. Loud setup errors for a wrong-family sensor, a missing helper, or the not-yet-built Entity mode. The Entity mode is reserved in the same blueprint for a later beta. |

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>). This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.

You will find the full YAML, this README, and the one-click import badge at the [Thinking Home blueprints repository](https://github.com/TheThinkingHome/Automations).
