# Sentinel Notify (Alpha)

A Home Assistant automation blueprint that turns a Battery Sentinel or Entity Sentinel into smart, change-aware notifications: it tells you what is wrong, by name, and does not spam your notifications every cycle.

## What This Solves

The Sentinels produce a sensor with a rich set of attributes: not just a count, but a list of exactly which batteries are low (with percentage, battery type, and area) or which entities have gone quiet (with the reason and how long). That detail is the whole point of building a sensor instead of a one-shot notifier. But a sensor does not notify you on its own; it needs something that reads it and decides when and how to alert you.

This is a companion to the two Sentinel sensor blueprints, and does nothing on its own. Build a Sentinel first, it creates the sensor, then build this to notify it:

- [**Battery Sentinel**](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/battery_sentinel.md) counts the batteries running low.
- [**Entity Sentinel**](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/entity_sentinel.md) catches entities that have gone quiet, unavailable or frozen at their last value.

Because it reads the structured attributes, it can do two things the common notifier blueprints cannot. It can name what is wrong, "Master Bath Motion (12%, CR2450)", not just "3 batteries low." It can notify you only when the situation has changed, so it does not repeat every cycle about the same three batteries or entities you already know about. This companion comes from **The Thinking Home** at [xeazy.com](https://xeazy.com). The full design and worked examples are in the [article](https://xeazy.com/battery-entity-sentinel-blueprints/).

## Built on Request, and Why It Is an Alpha

Every other blueprint from The Thinking Home grew out of my own house. The sensors and automations behind them ran on my system for years before I shared them, so the reliability was already settled. I generalized what already worked: made it configurable, documented it, smoothed the edges, and handed over something I already trusted.

This one is different. It did not run quietly for years in my walls. It was asked for, by the community, for a problem that existing tools cannot solve. So I built it: imagined, drafted, edited, reimagined, debugged, and am still testing it as hard as I can in one house and on an adversarial test suite.

That is why this is an **alpha** release. It is genuinely useful today, and I run it on my own system. But if you use it, you are also helping to test and refine it. Real homes are more varied than any one setup, and the edge cases that matter most are the ones I did not imagine and cannot produce. If you hit something, the [community thread](https://xeazy.com/logbook/d/42-the-battery-entity-sentinel-blueprints) is where it gets found and fixed. That is the deal: I have done my best to build it, and the community's use is what will make it solid.

> **The Entity mode is the newest and least-proven part.** The Battery mode has had the most testing and is the more settled of the two. The Entity mode was built most recently, has had the least time on real systems, and is the part most likely to change between releases in its wording, its behavior, and its options. If you run Entity mode, treat it as the leading edge of this alpha: it is where bugs are most likely to surface and where your feedback has the most influence on what it becomes. Expect it to move.

## How it Works

This companion watches one Sentinel family at a time. You set a mode, _Battery or Entity_, and point it at sensors from that family. If you run both Sentinels, build two copies of this companion. They behave differently, batteries are sparse and slow, entities are denser and twitchier, and keeping them separate makes each one simpler to configure and its notifications cleaner to read.

It does not poll. It re-evaluates the moment a watched Sentinel sensor changes, and once when Home Assistant starts. There is no check interval to tune, and no idle timer running in the background. The only scheduled event is the optional daily reminder, described below.

## What It Does

- **Reads the Sentinel's attributes** and reports the actual devices. For batteries: the name, percentage, battery type, and area; a binary low-battery sensor (which has no percentage) is reported as "(low)". For entities: the name, the raw reason it is flagged, and how long since it last reported, with the area.
- **Notifies only on change.** It reacts when a watched sensor changes, and sends a notification only when the set of flagged items is different from what it last told you about. The same unchanged situation stays quiet.
- **Combines several sensors into one report.** Point it at all your Battery Sentinel sensors (or all your Entity Sentinel sensors) and a single notification covers them all.
- **De-duplicates an item flagged by two sensors.** If overlapping Sentinel scopes flag the same thing twice, it collapses to one entry. In Battery mode, offline wins over low. In Entity mode, a genuinely-gone reason (`unavailable`, `unknown`, `missing`, `never_reported`) wins over `frozen`.
- **Sends where you want.** Push to any number of mobile devices, and optionally a persistent notification in the Home Assistant UI, replaced in place so it never stacks.
- **Priority is per device**. Two lists, high and normal. High bypasses Do Not Disturb (alarm sound on Android, time-sensitive on iOS); normal is standard delivery.
- **Optional quiet hours**. Off by default. Hold normal-priority pushes during a daily window, for example overnight; high-priority pushes still come through. The persistent card keeps updating. Pair it with the daily reminder's overnight mode, below, to hear in the morning about a change that was held.
- **Optional daily reminder, in one of two modes**. Off by default. Overnight changes: fires at a time you set, only when something changed during your quiet hours, and lists everything currently flagged; a quiet night stays silent. Daily summary: fires every day at that time, with the flagged list, or an all-clear when nothing is flagged. Either mode reuses the same notification, so it refreshes the existing one rather than stacking.
- **Rides out a restart, on both streams.** A source Sentinel goes briefly unavailable during a Home Assistant reboot, and reports an empty flagged list while inside its own startup grace. The "Sentinel not working" alert is debounced so a normal restart does not fire a false alarm, and the whole run is held while the sources settle after a reboot, so the change-detection memory survives and known items are not re-announced as new. Optionally, press your Sentinels' refresh button to confirm a recovery fast.
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

When you create the helper, open advanced options and set the maximum length to 255. The default of 100 is too short: the companion stores two fingerprints, a held-overnight flag, and a timestamp, which can run past 100 characters when something is flagged, and if the helper is too short the write fails silently and change-detection stops working. Setting it to 255 is all the helper needs.

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
4. **Notification Targets and Style**, where and how to send, including the high- and normal-priority lists.
5. **Quiet Hours**, an optional window that holds normal-priority pushes overnight.
6. **Daily Reminder**, the optional once-a-day send, its mode, and its time.
7. **Advanced**, the broken-Sentinel debounce, an optional source refresh, and debug logging.

The Battery Settings section applies only to Battery mode; in Entity mode, leave it alone. Entity mode has no extra options of its own, the sensor decides what is flagged, and this companion reports it.

## Parameters

| Input | Required | Default | What it does |
| --- | --- | --- | --- |
| Sentinel family | Yes | Battery | Battery or Entity. One family per copy. |
| Sentinel sensors | Yes | none | One or more Sentinel sensors of the chosen family. Combined into one report. Pointing at the wrong family stops with a setup error. |
| Memory helper | Yes | none | The `input_text` the companion uses to remember what it last reported. One per copy. |
| Also report offline batteries | No | off | Battery mode only. When on, also reports batteries that are currently unavailable, alongside the low ones. |
| High-priority push targets | No | none | The mobile_app notify action(s) to receive high-priority pushes, one per line, in the form `notify.mobile_app_yourphone` (the action name, not the `notify.yourphone` entity, which cannot carry the priority and sound payload). High bypasses Do Not Disturb and pierces quiet hours. Find the exact name in Developer Tools, Actions, search mobile_app. |
| Normal-priority push targets | No | none | The mobile_app notify action(s) to receive normal-priority pushes, same form. Normal is standard delivery and respects quiet hours; use the daily reminder's overnight mode to hear in the morning about a change that was held. A target in both lists is treated as high. Leave both lists empty to use only the persistent notification. |
| Create a persistent notification | No | on | Also creates or updates a persistent notification in the HA UI, replaced in place. |
| Notification tag | No | `sentinel_notify` | The tag that replaces this companion's push in place. Give each copy a distinct tag if several push to one device. |
| Enable quiet hours | No | off | When on, normal-priority pushes are held during the window below; high-priority pushes still come through. The persistent card and memory keep updating. Nothing fires when the window ends. |
| Quiet hours start | No | `22:00:00` | When the quiet window begins. A start later than the end is treated as crossing midnight. |
| Quiet hours end | No | `08:00:00` | When the quiet window ends. Nothing fires at this time; it only defines the window. |
| Daily reminder mode | No | None | What the once-a-day send does. None: nothing scheduled ever fires. Overnight changes: fires only when a change occurred during quiet hours, listing everything currently flagged; requires quiet hours, and the time should be at or after the quiet-hours end. Daily summary: fires every day, with the flagged list or an all-clear when nothing is flagged (persistent card plus high-priority targets only). |
| Reminder time | No | `08:00:00` | The time of day the reminder fires, when the mode is not None. For overnight mode, set it at or after the quiet-hours end. |
| Broken-Sentinel debounce (seconds) | No | `240` | How long a source must stay unavailable before it is reported as not working, and how long a run is held while sources settle after a reboot. A setup error or missing source is reported immediately regardless. |
| Source refresh button (optional) | No | none | An `input_button` that re-evaluates your source Sentinels, the same one you set as their refresh button. When set, an unavailable source is prompted to re-check and confirm recovery fast. Left unset, nothing is pressed. |
| Source refresh timeout (seconds) | No | `15` | The most time to wait for sources to recover after a refresh press. The wait ends as soon as every source is healthy; this cap only matters on a slow, large system. Ignored when no refresh button is set. |
| Debug logging | No | off | Writes one diagnostic line per check. Needs a `logger` entry at info level to appear. |

## How "Notify Only on Change" Works

The companion keeps a small fingerprint of what it last reported in the memory helper. When a watched sensor changes, it builds the same kind of fingerprint from what the Sentinel is flagging right now, sorted so the order never matters, and compares the two. If they differ, the situation has changed: it sends a notification listing everything currently flagged, and stores the new fingerprint. If they match, it stays silent.

A battery dropping below the threshold, or one being replaced, changes the fingerprint and you hear about it. The same three batteries sitting low for a week do not re-notify you every cycle, you were told once. In Entity mode, the same applies: an entity going quiet, or coming back, changes the fingerprint; an entity that stays quiet does not re-notify. The reason is part of the fingerprint, so an entity that shifts from `frozen` to `unavailable` counts as a change and you hear about it.

By design, in Battery mode, the change is tracked at the level of which batteries are low, not their exact percentage. A battery that keeps draining while staying below the threshold (say 15% down to 5%) will not send a fresh alert, you were already told it is low. This is deliberate: tracking the exact percentage would re-notify on every reading and bring back the flood the change detection exists to prevent. If you want to be nudged about something you have not yet dealt with, use the daily reminder in its daily summary mode.

Because the notification uses a tag, even when it does fire, it replaces the previous one in place rather than stacking a pile of alerts.

If one of the source Sentinels is itself broken (its uptime sensor was deleted, or it reports a setup error), the companion sends a separate "Sentinel not working" notification, on its own tag, so it does not interfere with the main report. A broken monitor never silences the healthy ones: the items on the working Sentinels are still reported in the same run. A source that has simply gone unavailable is held for a short debounce first, so a Home Assistant restart does not fire a false alarm; see "Riding Out a Restart" below.

## Riding Out a Restart

A Home Assistant restart disturbs both alert streams, and both are guarded.

A source Sentinel goes unavailable for a minute or two every time Home Assistant restarts, as its template entity is unloaded and reloaded. Without a guard, the "Sentinel not working" alert would fire on every reboot, and a high-priority target would be woken by it overnight. So the unavailable case is debounced. A source is only reported as not working once it has stayed unavailable for at least the **broken-Sentinel debounce** (an Advanced input, default 240 seconds, four minutes). A normal restart recovers well inside that window and never fires; a source that is genuinely dead stays unavailable past it and is reported correctly. The default of 240 matches the startup grace the Sentinel sensors use and clears a normal restart with room to spare. Lower it if you want a faster broken-Sentinel alert and can accept a restart occasionally tripping it; raise it if an unstable system still trips it during reboots. A setup error, an `ok:false`, or a genuinely missing source is not debounced, since a reboot neither causes nor cures those, and is reported immediately.

The normal alert stream is guarded too. Inside their startup grace after a reboot, the source Sentinels report an empty flagged list. Trusting that empty list would wipe the change-detection memory, and the re-detection of the same entities minutes later would arrive as false "new problem" pushes. So while any source has rebooted within the same debounce window, the whole run is held: no memory write, no notification, no card change. The pre-restart memory survives, the re-detected set compares equal, and the reboot passes silently. A genuinely new failure during the window is delayed by at most the settle time. A change that was held for quiet hours before the reboot also survives it, so the overnight reminder still delivers it in the morning.

Optionally, you can point the **source refresh button** input at the same `input_button` your Sentinels use as their refresh button (one shared button refreshes a whole family). When a source reads unavailable, the companion presses it to make the sources re-evaluate at once, then waits for the state to recover before deciding, so a source that had only gone stale is confirmed healthy in seconds rather than waiting for its next scheduled scan. The wait ends the instant every source is healthy and is capped by the **source refresh timeout** (default 15 seconds), which only matters on a slow hub with a large mesh that needs longer to re-evaluate. The button is optional: left unset, nothing is pressed and the debounce alone handles restarts.

## The Daily Reminder

Change-detection tells you the moment something changes. The daily reminder is the one scheduled send, and its mode decides what that send means. Off (None) by default.

**Overnight changes** is the partner to quiet hours. When a change fires during the quiet window, your normal-priority devices are held; this mode delivers that held report at the time you set, listing everything currently flagged. If nothing changed during the window, the morning stays silent. A problem that appeared and cleared on its own overnight also stays silent, and a list that partly cleared reports only what remains. A high-priority phone that already got the alert live overnight simply sees the card refresh under the same tag, rather than a second alert. This mode requires quiet hours to be enabled; with quiet hours off, nothing is ever held, so it behaves as None. Set the reminder time at or after the quiet-hours end.

**Daily summary** is the once-a-day nudge for things you keep meaning to deal with, a battery you have not replaced, or an entity still offline. Every day at the time you set, it sends the current report if anything is flagged. When nothing is flagged, it sends an all-clear instead, so a clean fleet shows reassurance rather than silence: a persistent "all clear" card in the UI, plus a push to high-priority targets only, at standard delivery. Normal-priority targets never receive the all-clear. A high-priority phone in this mode gets exactly one guaranteed touch per day, the flagged list or the all-clear.

In either mode, the send reuses the same notification tag, so it refreshes the card already on your phone rather than stacking a second one, and if you had dismissed it, a fresh one appears. It delivers to normal-priority targets even if its time falls inside the quiet window, since you chose that time explicitly. It also re-raises the "Sentinel not working" alert if a source is still broken.

The reminder never disturbs change-detection. The stored fingerprints are updated only on a real change, never by a reminder, so a daily send cannot confuse the "notify only on change" logic.

## Quiet Hours

Quiet hours hold your phone alerts during a window you set, for example 10 at night to 8 in the morning, so a low battery at 2 AM does not wake you. Off by default.

It is built around the per-device priority lists. A normal-priority target is held during the window; a high-priority target pierces it and is alerted at any hour. So you can keep your own phone on the high list to be woken by something genuinely urgent, while the rest of the household, on the normal list, waits until morning.

Two things keep working through the window: the persistent notification still updates silently, so the dashboard card is always current, and the change-detection memory still tracks, so nothing is lost. Only the normal-priority phone push waits.

Nothing fires when the window ends. To hear in the morning about a change that was held overnight, set the daily reminder to its **overnight changes** mode with a time at or after the quiet-hours end; that is the delivery half of quiet hours. With the reminder set to None, a change held overnight reaches your normal-priority devices only on the next change, though the persistent card has shown it all along.

The window may cross midnight: a start later than the end (22:00 to 08:00) is read as an overnight window.

## Keeping It Calm

The companion is only as quiet as the sensor you point it at. For batteries this is rarely an issue, batteries cross the threshold one at a time, so notifications are naturally infrequent. The advice matters more for the Entity mode: watch the handful of things whose silence is a real problem, not every entity in the house, because a sensor flagging dozens of flapping entities will notify you about churn no matter how good the change detection is. A tight, deliberate scope on the Sentinel is what keeps the notifications meaningful.

## Putting It to Work

The simplest setup: Battery mode, point it at your Battery Sentinel sensor, select a memory helper, add your phone to the normal-priority list, and leave the rest at defaults. You will get a notification when a battery drops below the threshold, named and with its battery type, and another when you replace it, with nothing in between.

For Entity mode, point it at your Entity Sentinel sensor with its own dedicated memory helper. You will get a notification naming each entity that has gone quiet and why, the moment the set changes, and again when one comes back.

For a quieter, dashboard-only setup, leave both priority lists empty and keep the persistent notification on; the report lives in the Home Assistant UI and updates itself, with no phone alerts at all.

For the "remind me about what I keep ignoring" setup, set the daily reminder mode to daily summary and pick a time. Anything still flagged nudges you once a day until you deal with it, and on a clean day the same run confirms everything is healthy with an all-clear.

For the "do not wake me" setup, put your own phone on the high-priority list and everyone else on the normal list, enable quiet hours for overnight, and set the daily reminder to overnight changes with a time at the end of the window. Urgent alerts still reach your phone at any hour; a change that appeared overnight reaches the rest of the household in the morning; and a quiet night stays quiet for everyone.

More worked examples are in the article: <https://xeazy.com/battery-entity-sentinel-blueprints/>

## Changelog

| Version | Notes |
| --- | --- |
| 1.0.0-alpha.7.1 | One scheduled send instead of two. The daily reminder and a separate re-send at the quiet-hours end time overlapped in purpose and could both fire in one morning; the quiet-end push also re-sent long-known items every day just because the window closed. The quiet-hours-end trigger is removed (the end time still defines the window; nothing fires at it) and the daily reminder becomes a three-way mode select replacing two toggles: None (default, notify only on change), Overnight changes (fires at the reminder time only when a change occurred during quiet hours, listing the full currently flagged set; silent after a quiet night; behaves as None when quiet hours is off), and Daily summary (fires every day; the flagged list, or an all-clear when nothing is flagged, sent as a persistent card plus a push to high-priority targets only at standard delivery). The overnight mode is driven by a held flag stored in the memory helper; an existing helper value upgrades in place with no manual reset. The reminder now delivers to normal-priority targets even when its time falls inside quiet hours. The separate all-clear toggle is absorbed into the daily summary mode. **Breaking input change**: `Enable the daily reminder` and `Send a daily all-clear` are replaced by `Daily reminder mode`; open each existing automation, pick a mode, and save. |
| 1.0.0-alpha.7 | Ride out the restart on the normal alert stream. During a reboot the source Sentinels report an empty flagged list while inside their startup grace; previously the companion trusted that empty list, wiped its memory, and re-announced the same known items minutes later as new failures. Now the whole run is held, no memory write, no notification, no card change, while any source has rebooted within the broken-Sentinel debounce window, so the pre-restart memory survives and a reboot passes silently. A genuinely new failure during the window is delayed by at most the settle time. Reads the boot_time attribute the Sentinels already expose; no new input, non-breaking. |
| 1.0.0-alpha.6.1 | Two additions that stop a Home Assistant restart from firing a false "Sentinel not working" alert, both non-breaking. Broken-Sentinel debounce: a source Sentinel goes unavailable for a minute or two during any restart, and the broken-Sentinel alert previously fired on every reboot, piercing quiet hours to a phone overnight. The unavailable case is now debounced: a source is reported not working only after it has been unavailable for at least the debounce (new Advanced input `Broken-Sentinel debounce`, default 240 seconds). A restart recovers inside that window and never fires; a genuinely dead source stays unavailable past it and still fires. A setup error, an `ok:false`, or a genuinely missing source is reported immediately, not debounced. Optional active source refresh: a new optional `Source refresh button` input (default none). When set and a source reads unavailable, the companion presses that button (the same one your Sentinels use as their refresh button) and waits briefly, capped by `Source refresh timeout` (default 15 seconds), so a source that had only gone stale is confirmed healthy in seconds rather than at its next scheduled scan. Left unset, the debounce alone handles restarts. This shifts the timing of an existing alert and changes no saved input, so an existing automation keeps working and simply gets quiet reboots. |
| 1.0.0-alpha.5 | Three features. Quiet hours: an optional daily window that holds normal-priority pushes; high-priority pushes pierce it. The persistent card and memory keep updating through the window, and at the end time anything still flagged is re-sent, so a problem raised overnight reaches you in the morning and one that cleared does not. Handles a window crossing midnight. Per-device priority: the single push-target list and the global priority selector are replaced by two lists, high-priority and normal-priority push targets, so priority is set per device; a target in both is treated as high. The "low" (silent) option is removed. This is a breaking input change, acceptable while the blueprint is pre-release alpha. Daily all-clear: an optional persistent-only "all clear" card on the daily reminder when nothing is flagged, so a clean fleet shows reassurance; it never pushes. The change-detection engine, memory format, and message builders are unchanged. |
| 1.0.0-alpha.4 | Cleanup, no behavior change. Removed two redundant always-true wrappers around the push actions (`continue_on_error` on the action already handles a dead target), and fixed a stray double apostrophe in the memory-helper description. |
| 1.0.0-alpha.3 | Fixed push targets. The input now takes the mobile_app notify action name (`notify.mobile_app_yourphone`) as text, the form that carries the full priority and sound payload; the previous notify-entity form (`notify.yourphone`) could not be called as an action and raised an "unknown action" error on real systems. Also dropped a now-meaningless per-target availability check; `continue_on_error` already makes a dead target non-fatal. |
| 1.0.0-alpha.2 | Added the Entity mode: reads Entity Sentinel sensors and reports the entities that have gone quiet, by name, with the raw reason (`unavailable`, `unknown`, `missing`, `frozen`, `never_reported`) and how long since each last reported, and the area. When two overlapping Entity Sentinels flag the same entity, a genuinely-gone reason wins over `frozen`. Also in this release: removed the minute-based check interval and the elapsed-time re-remind in favor of pure event-driven change-detection (re-evaluates when a watched sensor changes, and at Home Assistant start, with no polling), and added an optional daily reminder at a configurable time of day that re-sends the current report if anything is still flagged, without disturbing change-detection. The Entity mode is the newest and least-proven part and is most likely to change. |
| 1.0.0-alpha.1 | Initial build. Battery mode: reads Battery Sentinel sensors and reports low batteries (and optionally offline ones) by name with percentage, battery type, and area; binary low-battery sensors are reported without a percentage. Notifies only when the flagged set changes, using a sorted, hashed fingerprint stored in one `input_text` helper, so it does not notify every cycle. A device reported by two overlapping-scope sensors is de-duplicated to one entry. A separate, change-detected "Sentinel not working" alert fires when a source Sentinel is in a setup error or has gone unavailable, without suppressing the battery alerts from the healthy sensors. Push to any number of mobile devices and/or a persistent notification, tag-replaced in place, with priority handling for iOS and Android. |

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>). This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.
