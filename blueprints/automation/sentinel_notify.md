# Sentinel Notify

A Home Assistant automation blueprint that turns a Battery Sentinel or Entity Sentinel into a live to-do list and addition-only notifications: every problem becomes an item on a list you can see, check off, and watch resolve, and your phone hears only about what is new.

## What This Solves

The Sentinels produce a sensor with a rich set of attributes: not just a count, but a list of exactly which batteries are low (with percentage, battery type, and area) or which entities have gone quiet (with the reason). That detail is the whole point of building a sensor instead of a one-shot notifier. But a sensor does not notify you on its own; it needs something that reads it and decides when and how to alert you.

This is a companion to the two Sentinel sensor blueprints, and does nothing on its own. Build a Sentinel first, it creates the sensor, then build this to notify from it:

- [**Battery Sentinel**](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/battery_sentinel.md) counts the batteries running low.
- [**Entity Sentinel**](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/entity_sentinel.md) catches entities that have gone quiet, unavailable or frozen at their last value.

Because it reads the structured attributes, it can do three things the common notifier blueprints cannot. It can name what is wrong, "Master Bath Motion (12%, CR2450)", not just "3 batteries low." It can notify you only about what is **new**, so the three batteries you already know about never re-alert. And it keeps the whole picture as a to-do list: a shopping list for batteries, an outage board for entities, with a checkbox that actually means something. This companion comes from **The Thinking Home** at [xeazy.com](https://xeazy.com). The full design and worked examples are in the [article](https://xeazy.com/battery-entity-sentinel-blueprints/).

## Why 2.0, and Why It Is an Alpha

The 1.x engine remembered what it last reported as a hashed fingerprint inside an `input_text` helper, because a helper caps at 255 characters and a hash was the only way to fit. A hash can answer one question, did the set change, and nothing else. 2.0 moves the memory into a **Local To-do list**: the actual items, however many there are. That single change gives the engine real set arithmetic (it knows *what* changed, not just *that* something did), gives you the list as a living report, and deletes the 255-character wall for good.

The 2.0 engine is young but proven where it counts: it has passed an adversarial simulation suite of 49 scenario checks and external code review, and it runs live on my own system, both families, real notifications, through real nightly reboot cycles. What it has not had is months of varied homes, and real homes are more varied than any one setup. That is why it is still an **alpha**: if you run it, you are also helping test it, and the [community thread](https://xeazy.com/logbook/d/42-the-battery-entity-sentinel-blueprints) is where anything you hit gets found and fixed.

## How it Works

This companion watches one Sentinel family at a time. You set a mode, _Battery or Entity_, and point it at sensors from that family. If you run both Sentinels, build two copies of this companion. They behave differently, batteries are sparse and slow, entities are denser and twitchier, and keeping them separate makes each one simpler to configure and its notifications cleaner to read.

Every flagged device is one item on a dedicated to-do list. On each run the engine diffs what the Sentinels flag now against the items on the list: something new is added to the list and pushed to your phone by name; something recovered is removed from the list silently; everything already known stays quiet. The list is both the memory and the report.

It does not poll. It re-evaluates the moment a watched Sentinel sensor changes, when a source has been unavailable long enough to be declared broken, and once when Home Assistant starts. There is no check interval to tune. The only scheduled event is the optional daily reminder, described below.

## What It Does

- **Keeps one to-do item per flagged device.** For batteries: name, percentage, battery type, and area ("Front Door (14%, CR2450) - Entry"); a binary low-battery sensor is shown as "(low)". For entities: name, the raw reason, and area ("Front Door Lock (unavailable) - Entryway"). The item's due date is the moment it was detected, so the list shows how long each problem has been open.
- **Notifies only about what is new.** A brand-new device, or a known problem whose reason changed (a frozen entity escalating to unavailable), pushes by name: "1 new battery to check." Known items never re-alert, and a recovery removes its item without a sound.
- **The checkbox acknowledges.** Check an item off and it stays known (it will never re-notify), drops out of the daily summary, and re-arms by itself if the device recovers and later fails again. See "The To-do List" below.
- **Combines several sensors into one list.** Point it at all your Battery Sentinel sensors (or all your Entity Sentinel sensors) and a single list and notification stream covers them all.
- **De-duplicates a device flagged by two sensors.** In Battery mode, offline wins over low. In Entity mode, a genuinely-gone reason (`unavailable`, `unknown`, `missing`, `never_reported`) wins over `frozen`.
- **Sends where you want.** Push to any number of mobile devices, and optionally a persistent notification in the Home Assistant UI showing the current open items, replaced in place so it never stacks.
- **Priority is per device.** Two lists, high and normal. High bypasses Do Not Disturb (alarm sound on Android, time-sensitive on iOS); normal is standard delivery.
- **Optional quiet hours.** Off by default. Hold normal-priority pushes during a daily window; high-priority pushes still come through. The to-do list and the persistent card keep updating. Pair it with the daily reminder's overnight mode to hear in the morning about anything added overnight.
- **Optional daily reminder, in one of two modes.** Off by default. Overnight changes: fires only when something appeared during your quiet hours, and lists only those items, judged by each item's own detection time; a quiet night stays silent. Daily summary: fires every day, with the open items or an all-clear when nothing is flagged. Either way the send cannot be lost: the scheduled time is only a wake-up, the send itself is checked on every run and delivered exactly once per day, so a busy instant or a reboot at the wrong moment only delays it to the next run.
- **Rides out restarts and blips, on both streams.** The "Sentinel not working" alert is debounced so a reboot never fires it falsely, and fires on time when a source really dies. The whole run is held while sources settle after a reboot or blip briefly unavailable, so the to-do list can never be mass-deleted and re-announced by a restart.
- **Fails loud if set up wrong.** A wrong-family sensor, a missing or deleted to-do list, or a missing or deleted helper stops with a clear log message rather than failing quietly.

The full design and worked examples are in the article: <https://xeazy.com/battery-entity-sentinel-blueprints/>

## What the Entity Report Looks Like

In Entity mode, each flagged entity is reported with the raw reason it was flagged, exactly as the sensor classifies it, so you can tell a genuinely-gone device from a frozen one at a glance:

- **`unavailable`**, the entity's own state is `unavailable`.
- **`unknown`**, the entity's own state is `unknown`.
- **`missing`**, the entity does not exist (it was renamed, removed, or never created).
- **`frozen`**, the entity still has a value but has not reported a fresh one within the lookback window, it is stale, presenting as healthy while the device behind it has gone quiet.
- **`never_reported`**, the entity exists but has never produced a reading.

An item reads like "Front Door Lock (unavailable) - Entryway" or "Freezer Probe (frozen) - Kitchen"; how long it has been open shows through the item's date on the list. A new-problem push is titled "Entity Sentinel: 2 entities newly quiet"; a summary is titled "Entity Sentinel: 2 entities gone quiet."

## Before You Start: the Two Stores

This companion needs two things created first, one of each per copy.

**A dedicated Local To-do list.** This holds one item per flagged device: the companion's memory and your live report. Create it: Settings, Devices & Services, Add Integration, search **Local To-do**, and name it, for example "Battery Sentinel." You will select it in the blueprint below. Do not rename the list afterward. The companion only manages items it created (it marks them internally) and never touches anything else on the list, but a dedicated list is strongly recommended.

**An `input_text` helper.** This holds the companion's small internal state (the broken-Sentinel record and the quiet-hours held flag). Create it: Settings, Devices & Services, Helpers, Create Helper, Text. The default settings are fine, and a helper carried over from the 1.x release works as-is.

> **Critical: one dedicated list and one dedicated helper per copy.** Two copies (say a Battery one and an Entity one) must never share either store. Two copies on one list will fight over its items; two copies on one helper will overwrite each other's state.

## Import the Blueprint

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Fautomation%2Fsentinel_notify.yaml)

Or paste this URL into Settings, Automations & Scenes, Blueprints, Import Blueprint:

```
https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/automation/sentinel_notify.yaml
```

Your imported copy is a snapshot; re-import to pick up newer releases, and check the version number in the blueprint description. Upgrading from 1.x: re-import, then open each existing automation, select its new to-do list, and save. The helper carries over; the first run after upgrade announces everything currently flagged once, since the list starts empty, and the addition-only gate takes over from there.

## Setting Up the Automation

Unlike the Sentinels (which are template blueprints), this is an **automation** blueprint, so it sets up the easy way. After importing, go to Settings, Automations & Scenes, click Create Automation, and choose Sentinel Notify from the blueprint list. Fill in the form and save.

The form is organized into sections:

1. **Mode, Sensors, and Memory**, everything that defines this copy: the family, your Sentinel sensor(s), the to-do list, the helper, and the battery-only offline toggle.
2. **Notification Targets and Style**, where and how to send, including the high- and normal-priority lists.
3. **Quiet Hours**, an optional window that holds normal-priority pushes overnight.
4. **Daily Reminder**, the optional once-a-day send, its mode, and its time.
5. **Advanced**, the notification tag, the source debounce and settle window, an optional source refresh, and debug logging.

Entity mode has no options of its own, the sensor decides what is flagged and this companion reports it; the one battery-only toggle is labeled as such.

## Parameters

| Input | Required | Default | What it does |
| --- | --- | --- | --- |
| Sentinel family | Yes | Battery | Battery or Entity. One family per copy. |
| Sentinel sensors | Yes | none | One or more Sentinel sensors of the chosen family. Combined into one list. Pointing at the wrong family stops with a setup error. |
| To-do list | Yes | none | The dedicated Local To-do list that holds one item per flagged device. One per copy; do not rename it after selecting. |
| Memory helper | Yes | none | The `input_text` holding the companion's internal state: the broken-Sentinel record, the quiet-hours held flag, and the last daily-send date. One per copy. A 1.x helper works as-is. |
| Also report offline batteries | No | off | Battery mode only. When on, batteries that are currently unavailable are tracked alongside the low ones. |
| High-priority push targets | No | none | The mobile_app notify action(s) to receive high-priority pushes, one per line, in the form `notify.mobile_app_yourphone` (the action name, not the `notify.yourphone` entity, which cannot carry the priority and sound payload). High bypasses Do Not Disturb and pierces quiet hours. Find the exact name in Developer Tools, Actions, search mobile_app. |
| Normal-priority push targets | No | none | The mobile_app notify action(s) to receive normal-priority pushes, same form. Normal is standard delivery and respects quiet hours; use the daily reminder's overnight mode to hear in the morning about an addition that was held. A target in both lists is treated as high. Leave both lists empty for a list-and-card-only setup. |
| Create a persistent notification | No | on | Also maintains a persistent notification in the HA UI showing the current open items, replaced in place. |
| Notification tag | No | `sentinel_notify` | The tag that replaces this companion's push in place. Give each copy a distinct tag if several push to one device. |
| Enable quiet hours | No | off | When on, normal-priority pushes are held during the window below; high-priority pushes still come through. The list, the card, and the internal state keep updating. Nothing fires when the window ends. |
| Quiet hours start | No | `22:00:00` | When the quiet window begins. A start later than the end is treated as crossing midnight. |
| Quiet hours end | No | `08:00:00` | When the quiet window ends. Nothing fires at this time; it only defines the window. |
| Daily reminder mode | No | None | What the once-a-day send does. None: nothing scheduled ever fires. Overnight changes: fires only when something appeared during quiet hours, and lists only those items, judged by each item's own detection time (acknowledged ones excluded); requires quiet hours, and the time should be at or after the quiet-hours end. Daily summary: fires every day, with the open items or an all-clear when nothing is flagged at all (persistent card plus high-priority targets only). When every remaining item is acknowledged, the daily run stays silent rather than claiming all clear. |
| Reminder time | No | `08:00:00` | When the daily send is owed, when the mode is not None. The time is only a wake-up: the send is computed on every run and delivered once per day, so any time works, colliding with a sensor scan is harmless, and a send missed during a reboot is caught up by the next run. For overnight mode, set it at or after the quiet-hours end so the window has ended. |
| Source debounce and settle window (seconds) | No | `240` | One window, four jobs: how long a source must stay unavailable before it is reported broken (a dedicated trigger fires the report on time), how long runs are held while sources settle after a reboot or reload (judged live), how long after a source's own boot its reports count as still settling (so a source on a slow scan interval is never trusted stale; missing `boot_time` or `last_evaluated` holds the same way), and how long a brief source blip holds the run. A setup error or missing source is reported immediately regardless. |
| Source refresh button (optional) | No | none | An `input_button` that re-evaluates your source Sentinels, the same one you set as their refresh button. When set, a briefly unavailable source is prompted to re-check and confirm recovery fast. A source already down past the debounce is never pressed. Left unset, nothing is pressed. |
| Source refresh timeout (seconds) | No | `15` | The most time to wait for sources to recover after a refresh press. The wait ends as soon as every source is healthy. Ignored when no refresh button is set. |
| Debug logging | No | off | Writes one diagnostic line per check, carrying the diff counts. Needs a `logger` entry at info level to appear. |

## The To-do List: Memory, Report, and Acknowledgment

The list is the engine's memory, so it helps to know how the engine treats it.

**Each item carries three things.** The visible text is the human line. The item's description holds a small machine key the engine uses to recognize the device; do not edit it, and know that the engine owns the visible text too, a hand-edited line is rewritten on the item's next update. The item's date is when the problem was detected, so the list shows age natively.

**Notifications are addition-only.** The engine diffs the flagged devices against the list. A device not on the list is added and pushed by name. A known open device whose reason changed is updated and pushed, that is new information about an existing problem. A known device whose percentage or wording drifted is updated silently. A device no longer flagged has its item removed, silently: the recovery shows on the list and the card, and the daily summary confirms the clean state. This one rule is why a restart, a coordinator hiccup, or a staggered device recovery can never re-announce things you already know.

**The checkbox is yours, and it means acknowledged.** The companion never checks items off itself, so a checked item can mean only one thing: you saw it and dealt with it, or decided to live with it. An acknowledged item stays known (it will never re-notify), drops out of the daily summary and the card's open list, and has its text kept current silently in the background. When the device recovers, its item is removed like any other, which automatically re-arms it: if that device fails again later, that is a genuinely new incident, and it notifies. Unchecking an item simply returns it to the open list, no notification. Deleting an item by hand while the problem persists un-acknowledges it the hard way: the next run re-adds it and notifies.

**Two cautions.** Avoid the to-do card's "Remove completed items" menu action; it hand-deletes your acknowledged items, and the engine will re-add and re-notify any whose device is still down. And a rapidly flapping device (failing, recovering for a minute, failing again) will re-notify on each re-failure by design, since each is a new incident; the cure for a flapper is a longer grace window on the Sentinel watching it, not the notifier.

**Anything else on the list is invisible to the engine.** Items you add yourself are never touched, though a dedicated list keeps the report clean.

**Put it on a dashboard.** A standard to-do list card pointed at the list gives you the live report and the acknowledgment checkbox in one place. Keep completed items visible on the card, that is where your acknowledged entries live.

## Riding Out a Restart

A Home Assistant restart disturbs both alert streams, and both are guarded.

A source Sentinel goes unavailable for a minute or two every time Home Assistant restarts, as its template entity is unloaded and reloaded. Without a guard, the "Sentinel not working" alert would fire on every reboot, and a high-priority target would be woken by it overnight. So the unavailable case is debounced. A source is only reported as not working once it has stayed unavailable for at least the **source debounce and settle window** (an Advanced input, default 240 seconds, four minutes), and a dedicated trigger wakes the companion the moment that elapses, so a source that dies mid-day is reported on time rather than at the next unrelated event. A normal restart recovers well inside the window and never fires. A setup error, an `ok:false`, or a genuinely missing source is not debounced, since a reboot neither causes nor cures those, and is reported immediately.

The to-do list is guarded too. Inside their startup grace after a reboot, the source Sentinels report an empty flagged list. Trusting it would delete every item as a recovery and re-add the lot minutes later as new, notifying. So the whole run is held, no list change, no notification, no card change, while any source has rebooted within the debounce window, and likewise while any source is briefly unavailable inside that window (a coordinator restart, a template reload). The pre-restart list survives, the re-detected devices match their items, and the reboot passes silently. A genuinely new failure during the window is delayed by at most the settle time. One known limit: a source outage *longer* than the debounce drops that source's items, so its recovery re-announces them.

One more guard closes the reload case. When Home Assistant reloads template entities, or restarts, a source can pass through an instant where it looks alive but its attributes are not yet readable, and its first fresh answers come from inside its own startup grace. Three rules make that instant harmless: a source whose `boot_time` cannot be read is treated as rebooting rather than trusted; the settle state is re-judged live after any refresh press, so a run that woke grace-fresh sensors holds instead of believing their empty answers; and a run fired by a source coming back from unavailable is held outright, letting the source's next evaluation on its own settled cycle act instead. A fourth rule covers slow scan schedules: a source that has not re-evaluated since its own boot (its `last_evaluated` predates its `boot_time` plus the window) counts as settling however long ago that boot was, so a source that only looks every few hours can never feed the companion its stale grace-empty report; the first settled evaluation acts, and a daily send held this way is carried forward to it. The practical rule of thumb: after any restart, reload, or blip, the companion waits until the sources have demonstrably evaluated past their grace before believing anything. The one deliberate trade: a recovery that arrives as a source's return from unavailable is acted on one evaluation later than before.

Optionally, you can point the **source refresh button** input at the same `input_button` your Sentinels use as their refresh button (one shared button refreshes a whole family). When a source is briefly unavailable, the companion presses it and waits for recovery, bounded by the **source refresh timeout** (default 15 seconds), so a source that had only gone stale is confirmed healthy in seconds. A source already down past the debounce is never pressed or waited for, so one dead Sentinel cannot slow the rest. The button is optional: left unset, nothing is pressed and the debounce alone handles restarts.

## The Daily Reminder

Change detection tells you the moment something new appears. The daily reminder is the one scheduled send, and its mode decides what that send means. Off (None) by default.

**Overnight changes** is the partner to quiet hours, and it is a true summary of the window. When something is added during the quiet window, your normal-priority devices are held; this mode delivers at the time you set, listing **only the items that appeared during the window**, judged by each item's own detection time, so a week-old known problem never rides along in a morning summary (it shows only in the trailing "Also open" count). If nothing was added during the window, the morning stays silent. A problem that appeared and cleared on its own overnight, or one you acknowledged before morning, also stays silent. A high-priority phone that already got the alert live overnight simply sees the card refresh under the same tag. This mode requires quiet hours; with quiet hours off, nothing is ever held, so it behaves as None. Set the reminder time at or after the quiet-hours end.

**Daily summary** is the once-a-day nudge for things you keep meaning to deal with. Every day at the time you set, it sends the open items if there are any. When nothing is flagged at all, it sends an all-clear instead: a persistent card in the UI, plus a push to high-priority targets only, at standard delivery, so a high-priority phone gets exactly one guaranteed touch per day. When the only remaining items are ones you have acknowledged, the daily run stays silent, acknowledged is not fixed, and the companion will not claim all clear over a problem you have merely accepted.

**The send cannot be lost.** The scheduled time is only a wake-up. Whether the day's send is owed is computed on every run, and whichever run first finds it owed performs it and records the date, exactly once per day. So the reminder time may sit on the same instant as a sensor's scan without a conflict, a send missed because Home Assistant was rebooting at that moment is caught up by the next run, and a run that both owes the send and carries a brand-new detection merges the two into one notification rather than sending twice.

In either mode, the send reuses the same notification tag, delivers to normal-priority targets even if its time falls inside the quiet window (you chose that time explicitly), and re-raises the "Sentinel not working" alert if a source is still broken. The reminder never disturbs change detection.

## Quiet Hours

Quiet hours hold your phone alerts during a window you set, for example 10 at night to 8 in the morning, so a low battery at 2 AM does not wake you. Off by default.

It is built around the per-device priority lists. A normal-priority target is held during the window; a high-priority target pierces it and is alerted at any hour. So you can keep your own phone on the high list to be woken by something genuinely urgent, while the rest of the household, on the normal list, waits until morning.

Three things keep working through the window: the to-do list updates, the persistent card updates, and the internal state tracks, so nothing is lost. Only the normal-priority phone push waits.

Nothing fires when the window ends. To hear in the morning about something added overnight, set the daily reminder to its **overnight changes** mode with a time at or after the quiet-hours end; that is the delivery half of quiet hours. With the reminder set to None, an overnight addition reaches your normal-priority devices only on the next new item, though the list and card have shown it all along.

The window may cross midnight: a start later than the end (22:00 to 08:00) is read as an overnight window.

## Keeping It Calm

The companion is only as quiet as the sensor you point it at. For batteries this is rarely an issue, batteries cross the threshold one at a time. The advice matters more for the Entity mode: watch the handful of things whose silence is a real problem, not every entity in the house. The addition-only gate means a stable set of problems is completely silent, but a device that flaps, failing and recovering repeatedly, notifies on each re-failure, because each genuinely is a new failure. The cure for a flapper is a longer grace window on the Sentinel watching it. A tight, deliberate scope upstream is what keeps the notifications meaningful.

## Putting It to Work

The simplest setup: Battery mode, point it at your Battery Sentinel sensor, its own to-do list, its own helper, add your phone to the normal-priority list, and leave the rest at defaults. You will get a notification when a battery drops below the threshold, named and with its battery type, the item appears on the list, and when you replace the battery the item quietly disappears.

For Entity mode, the same with its own list and helper. You will get a notification naming each entity that has newly gone quiet and why, and the list is your live outage board.

For the shopping-list setup, put a to-do list card for the Battery list on a dashboard, and take the phone with you to the store; every line carries the battery type.

For a list-and-card-only setup, leave both priority lists empty and keep the persistent notification on; the report lives on the list and in the HA UI, with no phone alerts at all.

For the "remind me about what I keep ignoring" setup, set the daily reminder mode to daily summary and pick a time. Anything open nudges you once a day. Check off the items you have decided to live with, and they stop nudging until they resolve; on a fully clean day, the same run confirms everything is healthy with an all-clear.

For the "do not wake me" setup, put your own phone on the high-priority list and everyone else on the normal list, enable quiet hours for overnight, and set the daily reminder to overnight changes with a time at the end of the window. Urgent alerts still reach your phone at any hour; something new overnight reaches the rest of the household in the morning; and a quiet night stays quiet for everyone.

More worked examples are in the article: <https://xeazy.com/battery-entity-sentinel-blueprints/>

## Changelog

| Version | Notes |
| --- | --- |
| 2.0.0-alpha.9.4 | Debug logging only; no engine or input change. The result summary log line now names the entities in its list-mutating fields, not just their counts: `additions`, `kind_changed`, `silent_updates`, and `recovered` each print a bracketed `[entity_id, ...]` list when non-empty (and nothing extra when zero, so idle runs read as before). Prompted by a live incident where a frozen entity was flagged overnight and silently recovered at the next scan: the recovery was logged, but as `recovered: 1` with no name, so identifying which entity left the list meant cross-referencing the source sensor's own log. The notifier's stream is now self-documenting for every list change. |
| 2.0.0-alpha.9.3 | The stale-source guard, from a live near-miss. A source on a slow scan interval had not re-evaluated since its own boot, so its last report was the honest grace-empty one; the reminder duty fires on a time trigger, and by its time the boot sat outside the settle window, so the duty would have trusted the stale empty report, wiped every item as a recovery, and posted a false all-clear. New rule in both the top-of-run settle gate and the post-refresh live recheck: a source whose `last_evaluated` predates its own `boot_time` plus the window counts as settling, whatever its scan interval; a source missing `last_evaluated` while looking alive is held the same way. A held duty run stamps nothing, so the daily send carries forward to the first settled run, which sweeps and reports in one truthful shot. No input changes. |
| 2.0.0-alpha.9.2 | The reload guard, from a live failure: during a restart plus template reload, the sources passed through an attribute-less instant, the settle gate found no boot_time to read and concluded nothing was rebooting, the blip path's own refresh press woke the freshly recreated sensors into their startup grace, and their grace-empty answers deleted every item as a recovery. Three independent holds close the hole: a source that looks alive but has no readable `boot_time` counts as settling (sources that are unavailable, in setup_error, or `ok: false` are exempt, so the broken-Sentinel alert still fires); the settle state is re-judged live after any refresh, alongside the blip state; and a run fired by a source returning from unavailable or unknown is held outright, its source's next settled evaluation acting instead. A recovery arriving as a from-unavailable transition is now acted on one evaluation later, a deliberate trade. No input changes. |
| 2.0.0-alpha.9.1 | Form reorganization only; the engine is untouched and no input keys changed, so existing automations are unaffected. The one-input Mode section and the collapsed Battery Settings section are dissolved into a single "Mode, Sensors, and Memory" section holding everything that defines a copy; the notification tag moved to Advanced, where its only real audience lives; and the heavier input descriptions were trimmed by roughly a third with no rule removed. |
| 2.0.0-alpha.9 | The reliable reminder, two fixes from the first live morning. The daily send is now a duty, not a trigger: the scheduled time is only a wake-up, and whether the send is owed is computed on every run against a delivery date kept in the helper, so a reminder whose time collides with a sensor scan is delivered by the colliding run itself, any time is valid on or off the scan grid, and a send missed during a reboot is caught up by the next run, exactly once per day. And the overnight mode became a true summary of the quiet window: it lists only the open items whose own detection time falls inside the just-ended window (midnight crossing handled), so a long-known problem never rides along in a morning delivery; overnight arrivals that self-cleared or were acknowledged before morning stay silent, and a send that coincides with a brand-new detection merges into one notification under a dedicated overnight title. No input changes; existing automations upgrade in place, and an older helper value is read as never-delivered, so the first run past the reminder time performs that day's send once. |
| 2.0.0-alpha.8 | The to-do rebuild. The item memory moves from a hashed fingerprint in an `input_text` to a dedicated Local To-do list, one item per flagged device: the visible text is the report line, the description carries the machine identity, the date is the detection time. Change detection becomes true set arithmetic with an addition-only gate: a notification names what is new (a new device, or a known problem whose reason changed) instead of re-listing the set; a recovery removes its item silently; a restart or staggered device refill can never re-announce known items. The checkbox is an acknowledgment: a checked item stays known, leaves the daily summary and the card's open list, is kept textually current in silence, and re-arms automatically when its device recovers and later fails again. Recovery always removes, never checks off; the checkmark belongs to the user. The `input_text` helper remains for the broken-Sentinel record and the quiet-hours held flag; a 1.x helper carries over as-is. New required input: the dedicated to-do list, one per copy. Items the companion did not create are never touched. Everything from 1.x carries over: the two alert streams, the on-time broken-source trigger, the restart and blip holds (now protecting the list from mass deletion), the recovery-candidate-only refresh, quiet hours with per-device priority, the three-mode daily reminder, and the loud setup errors, now covering a deleted to-do list as well. **Breaking**: this is a new engine; upgrading from 1.x requires creating and selecting a to-do list per copy, and the first run announces everything currently flagged once. |

The retired 1.x line and its full history remain in the repository's frozen 1.x files for reference.

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>). This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.
