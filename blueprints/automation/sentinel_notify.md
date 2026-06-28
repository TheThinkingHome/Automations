# Sentinel Notify (Alpha)

A Home Assistant automation blueprint that turns a Battery Sentinel or Entity Sentinel into smart, change-aware notifications: it tells you what is actually wrong, by name, and does not SPAM your notifications.

## What This Solves

The Sentinels produce a sensor with a rich set of attributes: not just a count, but a list of exactly which batteries are low (with percentage, battery type, and area) or which entities have gone quiet (with the reason and how long). That detail is the whole point of building a sensor instead of a one-shot notifier. But a sensor does not notify you on its own; you need something that reads it and decides when and how to alert you.

This is a companion to the two Sentinel sensor blueprints, and it does nothing on its own. Build a Sentinel first, it creates the sensor, then build this to notify from it:

- [**Battery Sentinel**](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/battery_sentinel.md) counts the batteries running low.
- [**Entity Sentinel**](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/entity_sentinel.md) catches entities that have gone quiet, unavailable or frozen at their last value.

Because it reads the structured attributes, it can do two things the common notifier blueprints cannot. It can name what is wrong, "Master Bath Motion (12%, CR2450)", not just "3 batteries low." And it can notify you only when the situation has changed, so it does not nag you every cycle about the same three batteries or entities you already know about. This companion comes from **The Thinking Home** at [xeazy.com](https://xeazy.com). The full design and worked examples are in the [article](https://xeazy.com/battery-entity-sentinel-blueprints/).

## Built on Request, and Why It Is an Alpha

Every other blueprint from The Thinking Home grew out of my own house. The sensors and automations behind them ran on my system for years before I shared them, so the reliability was already settled. My job was to generalize what already worked: make it configurable, document it, smooth the edges, and hand over something I already trusted.

This one is different. It did not run quietly for years in my walls. It was asked for, by the community, for a problem the existing tools cannot solve. So I built it: imagined, drafted, edited, reimagined, debugged, and am still testing it as hard as I can in one house and an adversarial test suite allows.

That is why this is an **alpha** release. It is genuinely useful today, and I run it on my own system. But if you use it, you are also helping test and refine it. Real homes are more varied than any one setup, and the edge cases that matter most are the ones I did not imagine and cannot produce. If you hit something, the [community thread](https://xeazy.com/logbook/d/42-the-battery-entity-sentinel-blueprints) is where it gets found and fixed. That is the deal: I have done my best to build it, and the community's use is what will make it solid.

> **The Entity mode is the newest and least-proven part.** The Battery mode has had the most testing and is the more settled of the two. The Entity mode was built most recently, has had the least time on real systems, and is the part most likely to change between releases, in its wording, its behavior, and its options. If you run Entity mode, treat it as the leading edge of this alpha: it is where bugs are most likely to surface and where your feedback has the most influence on what it becomes. Expect it to move.

## How it Works

This companion watches one Sentinel family at a time. You set a mode, _Battery or Entity_, and point it at sensors from that family. If you run both Sentinels, build two copies of this companion. They behave differently enough, batteries are sparse and slow, entities are denser and twitchier, that keeping them separate makes each one simpler to configure and its notifications cleaner to read.

It does not poll. It re-evaluates the moment a watched Sentinel sensor changes, and once when Home Assistant starts. There is no check interval to tune, and no idle timer running in the background. The only scheduled event is the optional daily reminder, described below.

## What It Does

- **Reads the Sentinel's attributes** and reports the actual devices. For batteries: the name, percentage, battery type, and area; a binary low-battery sensor (which has no percentage) is reported as "(low)". For entities: the name, the raw reason it is flagged, and how long since it last reported, with the area.
- **Notifies only on change.** It reacts when a watched sensor changes, and sends a notification only when the set of flagged items is different from what it last told you about. The same unchanged situation stays quiet.
- **Combines several sensors into one report.** Point it at all your Battery Sentinel sensors (or all your Entity Sentinel sensors) and a single notification covers them all.
- **De-duplicates an item flagged by two sensors.** If overlapping Sentinel scopes flag the same thing twice, it collapses to one entry. In Battery mode, offline wins over low. In Entity mode, a genuinely-gone reason (`unavailable`, `unknown`, `missing`, `never_reported`) wins over `frozen`.
- **Sends where you want.** Push to any number of mobile devices, and optionally a persistent notification in the Home Assistant UI, replaced in place so it never stacks.
- **Respects priority.** Normal, high (bypasses Do Not Disturb, alarm sound on Android, time-sensitive on iOS), or low (silent).
- **Optional daily reminder.** Off by default. If you want a nudge about items you have not dealt with, set a time of day and, once a day at that time, it re-sends the current report if anything is still flagged. It reuses the same notification, so it refreshes the existing one rather than stacking.
- **Fails loud if set up wrong.** A wrong-family sensor or a missing memory helper stops with a clear log message rather than failing quietly.

The full design and worked examples are in the article: <https://xeazy.com/battery-entity-sentinel-blueprints/>

## What the Entity Report Looks Like

In Entity mode, each flagged entity is reported with the raw reason it was flagged and how long since it last reported. The reasons are shown exactly as the sensor classifies them, so you can tell a genuinely-gone device from a frozen one at a glance:

- **`unavailable`**, the entity's own state is `unavailable`.
- **`unknown`**, the entity's own state is `unknown`.
- **`missing`**, the entity does not exist (it was renamed, removed, or never created).
- **`frozen`**, the entity still has a value but has not reported a fresh one within the lookback window, it is stale, presenting as healthy while the device behind it has gone quiet.
- **`never_reported`**, the entity exists but has never produced a reading.

A line reads like "Front Door Lock (unavailable, 10 minutes ago) - Entryway" or "Freezer Probe (frozen, 4 hours ago) - Kitchen". The title reads "Entity Sentinel: 2 entities gone quiet."

## Before You Start: the Memory Helper

This companion needs one helper to do its job: an `input_text` it uses to remember what it last reported, so it can notify you only on a change.

Create it first: Settings, Devices & Services, Helpers, Create Helper, Text. Give it any name, for example "Battery Notify Memory." You will select it in the blueprint below.

> **Critical: one dedicated helper per copy.** Every copy of this companion must have its own input_text helper. Two copies (say a Battery one and an Entity one) sharing a single helper will overwrite each other's memory and cross-fire endless notifications, each thinking the situation changed on every other check. If you build more than one copy, create a separate helper for each.

When you create the helper, open advanced options and set the maximum length to 255. The default of 100 is too short: the companion stores two fingerprints plus a marker, which can run past 100 characters when something is flagged, and if the helper is too short the write fails silently and change-detection stops working. Setting it to 255 is all the helper needs.

## Import the Blueprint

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Fautomation%2Fsentinel_notify.yaml)

Or paste this URL into Settings, Automations & Scenes, Blueprints, Import Blueprint:

```
https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/automation/sentinel_notify.yaml
```

## Setting Up the Automation

Unlike the Sentinels (which are template blueprints), this is an **automation** blueprint, so it sets up the easy way. After importing, go to Settings, Automations & Scenes, click Create Automation, and choose Sentinel Notify from the blueprint list. Fill in the form and save.

The form is organized into sections:

1. **Mode**, choose Battery or Entity.
2. **Sensors to Watch**, pick your Sentinel sensor(s) of the chosen family, and select the memory helper you created.
3. **Battery Settings**, an optional toggle used only in Battery mode.
4. **Notification Targets and Style**, where and how to send.
5. **Daily Reminder**, the optional once-a-day nudge and its time of day.
6. **Advanced**, debug logging.

The Battery Settings section applies only to Battery mode; in Entity mode, leave it alone. Entity mode has no extra options of its own, the sensor decides what is flagged, and this companion reports it.

## Parameters

| Input | Required | Default | What it does |
| --- | --- | --- | --- |
| Sentinel family | Yes | Battery | Battery or Entity. One family per copy. |
| Sentinel sensors | Yes | none | One or more Sentinel sensors of the chosen family. Combined into one report. Pointing at the wrong family stops with a setup error. |
| Memory helper | Yes | none | The `input_text` the companion uses to remember what it last reported. One per copy. |
| Also report offline batteries | No | off | Battery mode only. When on, also reports batteries that are currently unavailable, alongside the low ones. |
| Push notification targets | No | none | The mobile_app notify action(s) to push to, one per line, each in the form `notify.mobile_app_yourphone` (the action name, not the `notify.yourphone` entity, which cannot carry the priority and sound payload). Find the exact name in Developer Tools, Actions, search mobile_app. Leave empty to use only the persistent notification. |
| Create a persistent notification | No | on | Also creates or updates a persistent notification in the HA UI, replaced in place. |
| Priority | No | normal | Push urgency. High bypasses Do Not Disturb; low is silent. |
| Notification tag | No | `sentinel_notify` | The tag that replaces this companion's push in place. Give each copy a distinct tag if several push to one device. |
| Enable the daily reminder | No | off | When on, once a day at the time below, re-sends the current report if anything is still flagged. |
| Reminder time | No | `08:00:00` | The time of day the daily reminder fires, if enabled. |
| Debug logging | No | off | Writes one diagnostic line per check. Needs a `logger` entry at info level to appear. |

## How "Notify Only on Change" Works

The companion keeps a small fingerprint of what it last reported, in the memory helper. When a watched sensor changes, it builds the same kind of fingerprint from what the Sentinel is flagging right now, sorted so the order never matters, and compares the two. If they differ, the situation has changed: it sends a notification listing everything currently flagged, and stores the new fingerprint. If they match, it stays silent.

So a battery dropping below the threshold, or one being replaced, changes the fingerprint and you hear about it. The same three batteries sitting low for a week do not re-notify you every cycle, you were told once. In Entity mode, the same applies: an entity going quiet, or coming back, changes the fingerprint; an entity that stays quiet does not re-notify. The reason is part of the fingerprint, so an entity that shifts from `frozen` to `unavailable` counts as a change and you hear about it.

By design, in Battery mode the change is tracked at the level of which batteries are low, not their exact percentage. A battery that keeps draining while staying below the threshold (say 15% down to 5%) will not send a fresh alert, you were already told it is low. This is deliberate: tracking the exact percentage would re-notify on every reading and bring back the flood the change detection exists to prevent. If you want to be nudged about something you have not yet dealt with, use the daily reminder.

Because the notification uses a tag, even when it does fire it replaces the previous one in place rather than stacking a pile of alerts.

If one of the source Sentinels is itself broken (its uptime sensor was deleted, or it has gone unavailable), the companion sends a separate "Sentinel not working" notification, on its own tag, so it does not interfere with the main report. A broken monitor never silences the healthy ones: the items on the working Sentinels are still reported in the same run.

## The Daily Reminder

Change-detection tells you when something changes. The daily reminder is for the opposite case: something that has not changed, but that you keep meaning to deal with, a battery you have not replaced, an entity still offline. Off by default.

When you enable it and set a time, then once a day at that time the companion checks whether anything is still flagged. If so, it re-sends the current report. Because it reuses the same notification tag, it refreshes the card already on your phone rather than stacking a second one, and if you had dismissed it, a fresh one appears. If nothing is flagged at that time, the reminder does nothing.

The reminder only re-sends; it never disturbs change-detection. The stored fingerprint is updated only on a real change, never by a reminder, so a daily nudge cannot confuse the "notify only on change" logic.

## Keeping It Calm

The companion is only as quiet as the sensor you point it at. For batteries this is rarely an issue, batteries cross the threshold one at a time, so notifications are naturally infrequent. The advice matters more for the Entity mode: watch the handful of things whose silence is a real problem, not every entity in the house, because a sensor flagging dozens of flapping entities will notify you about churn no matter how good the change detection is. A tight, deliberate scope on the Sentinel is what keeps the notifications meaningful.

## Putting It to Work

The simplest setup: Battery mode, point it at your Battery Sentinel sensor, select a memory helper, add your phone as a push target, and leave the rest at defaults. You will get a notification when a battery drops below the threshold, named and with its battery type, and another when you replace it, with nothing in between.

For Entity mode, point it at your Entity Sentinel sensor with its own dedicated memory helper. You will get a notification naming each entity that has gone quiet and why, the moment the set changes, and again when one comes back.

For a quieter, dashboard-only setup, leave the push targets empty and keep the persistent notification on; the report lives in the Home Assistant UI and updates itself, with no phone alerts at all.

For the "remind me about what I keep ignoring" setup, enable the daily reminder and set a time, and anything still flagged nudges you once a day until you deal with it.

More worked examples are in the article: <https://xeazy.com/battery-entity-sentinel-blueprints/>

## Changelog

| Version | Notes |
| --- | --- |
| 1.0.0-alpha.4 | Cleanup, no behavior change. Removed two redundant always-true wrappers around the push actions (`continue_on_error` on the action already handles a dead target), and fixed a stray double apostrophe in the memory-helper description. |
| 1.0.0-alpha.3 | Fixed push targets. The input now takes the mobile_app notify action name (`notify.mobile_app_yourphone`) as text, the form that carries the full priority and sound payload; the previous notify-entity form (`notify.yourphone`) could not be called as an action and raised an "unknown action" error on real systems. Also dropped a now-meaningless per-target availability check; `continue_on_error` already makes a dead target non-fatal. |
| 1.0.0-alpha.2 | Added the Entity mode: reads Entity Sentinel sensors and reports the entities that have gone quiet, by name, with the raw reason (`unavailable`, `unknown`, `missing`, `frozen`, `never_reported`) and how long since each last reported, and the area. When two overlapping Entity Sentinels flag the same entity, a genuinely-gone reason wins over `frozen`. Also in this release: removed the minute-based check interval and the elapsed-time re-remind in favor of pure event-driven change-detection (re-evaluates when a watched sensor changes, and at Home Assistant start, with no polling), and added an optional daily reminder at a configurable time of day that re-sends the current report if anything is still flagged, without disturbing change-detection. The Entity mode is the newest and least-proven part and is most likely to change. |
| 1.0.0-alpha.1 | Initial build. Battery mode: reads Battery Sentinel sensors and reports low batteries (and optionally offline ones) by name with percentage, battery type, and area; binary low-battery sensors are reported without a percentage. Notifies only when the flagged set changes, using a sorted, hashed fingerprint stored in one `input_text` helper, so it does not notify every cycle. A device reported by two overlapping-scope sensors is de-duplicated to one entry. A separate, change-detected "Sentinel not working" alert fires when a source Sentinel is in a setup error or has gone unavailable, without suppressing the battery alerts from the healthy sensors. Push to any number of mobile devices and/or a persistent notification, tag-replaced in place, with priority handling for iOS and Android. |

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>). This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.

You will find the full YAML, this README, and the one-click import badge at the [Thinking Home blueprints repository](https://github.com/TheThinkingHome/Automations).
