# Sensor Watchdog (Beta)

A Home Assistant automation blueprint that watches a set of sensors for the three ways they fail in practice (offline, frozen, drifting toward wedged) and recovers them by cycling the smart plug they are plugged into. A timer helper tracks heartbeats so the freeze case gets caught instead of just the offline case, and an active time window keeps a quiet empty room from being mistaken for a frozen device.

An Aqara FP2 presence sensor talks over WiFi and roughly twice a week it locks up and stops reporting. Sometimes its entity drops to `unavailable`; sometimes the value just freezes wherever it happened to be, and Home Assistant has no way to know the device is dead until the next state change that never comes. A reboot of its smart plug fixes it every time. A Zigbee router plug at the back of the property occasionally drops its routing table and stops forwarding for the devices behind it; cycling its power restores the mesh within a minute. A WiFi camera with a known firmware bug needs a kick every few days. In each case the fix is the same trivial thing (turn the upstream plug off, wait ten seconds, turn it back on) and the only question is when to trigger it.

This blueprint answers that question three ways at once. It cycles the plug when the entity goes `unavailable` for longer than a tolerance window. It cycles the plug when the entity stops reporting fresh values, which catches the freeze case the unavailable detector misses. And it can fire a daily preventive cycle on a schedule, for devices known to drift slowly into a wedged state regardless of what their entity says. The detection is event-driven end to end: no time pattern polling, no scanning loops. A heartbeat timer is reset by reports from the monitored entities, and the recovery cycle fires when the timer expires.

Full write-up and the longer story behind the design: <https://xeazy.com/sensor-watchdog-blueprint/>  Questions and discussion: <https://xeazy.com/logbook/>

**Current version: 1.0.0-beta.** This blueprint is shipped as a public preview. The author is dogfooding it on three Aqara FP2 sensors at home and will not graduate it to 1.0.0 stable until that deployment has run clean for a sustained window. Use it on your own setup if you are comfortable troubleshooting; feedback on the forum is welcome.

## What You Get

- Three independent failure detectors firing into the same recovery cycle: an `unavailable` detector with a tolerance window, a freshness detector backed by a timer helper, and an optional daily scheduled cycle.
- A three-way mode selector that lets you choose how much event traffic the freshness detector creates: responsive (catches heartbeats, heaviest load), lighter (catches state changes only, much lighter), or minimal (skip the freshness detector entirely and rely on the unavailable detector plus the optional scheduled cycle).
- An optional active time window for the freshness detector, so a presence sensor in an empty room overnight is not flagged as frozen just because nothing in the room is changing. The unavailable detector and the scheduled cycle ignore the window; a sensor that has dropped offline is broken at 3am the same as it is at 3pm.
- An optional restart guard input. Point it at a sensor that goes truthy during your Home Assistant or router reboot window, and all three recovery branches will skip while it is active. Stops the blueprint from cycling plugs in response to the natural unavailability that a reboot creates.
- A self-recovery branch that turns the recovery switch back on if it is found off for longer than the recovery cycle would normally hold it. Protects against a dashboard click or a voice command that accidentally takes the upstream plug offline.
- Debug logging that you can turn on during setup and turn off in normal operation.

A monitored entity counts as "online" whenever it has any state value at all that is not `unavailable` or `unknown`. The freshness detector measures from the entity's `last_reported` timestamp in responsive mode (which updates on every report including same-value heartbeats) or from `last_changed` in lighter mode (which only advances when the value actually changes).

## Where This Fits

The closest hand-rolled alternative is a one-off `Unavailable - <device>` automation written separately for each flaky device. That works for the offline case and is straightforward to write, but it misses freezes (the entity is technically still reporting, just stuck) and it duplicates the same hundred lines of YAML once per device. Three flaky FP2s mean three near-identical automations to maintain. Move the same logic into a blueprint and the YAML lives once, with one instance per device.

Polling-based watchdogs (a `time_pattern` that fires every five minutes and checks freshness in a template condition) catch the freeze case but at the cost of running an automation every five minutes forever, on every Home Assistant install, regardless of whether anything is wrong. The event-driven design here only does work when an actual report comes in or an actual timer expires.

This blueprint sits where you have one or more devices that wedge often enough to matter, where a power cycle fixes them, and where you want the recovery to be hands-off and visible in the automation traces. If you only have one such device and you are comfortable maintaining a bespoke automation for it, the bespoke automation may be the lighter answer. Past two or three, the blueprint pays for itself.

## Requirements

Home Assistant 2024.10 or newer. The blueprint uses templated `enabled:` fields on triggers to gate the responsive, lighter, and scheduled triggers from a single mode selector; that capability landed in 2024.10. On older versions the blueprint refuses to import with a clear error.

If you use the responsive or lighter detection modes, you will also need to create a Timer helper to act as the heartbeat clock. Minimal mode does not need one.

## Step 1: Import the Blueprint

The quick way, one click:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTheThinkingHome%2FAutomations%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fsensor_watchdog.yaml)

Or by hand:

1. Go to Settings, then Automations & Scenes, then the Blueprints tab.
2. Click Import Blueprint at the bottom right.
3. Paste this URL and click Preview:

```
https://github.com/TheThinkingHome/Automations/blob/main/blueprints/automation/sensor_watchdog.yaml
```

4. Confirm and Import.

After import, the blueprint appears on the Blueprints page with a Create Automation button.

## Step 2: Create the Heartbeat Timer

Skip this step if you plan to use minimal mode.

A Timer helper is the clock the freshness detector reads from. Reports from the monitored entities reset it; if it ever expires, the blueprint declares the device frozen.

1. Go to Settings, then Devices & Services, then the Helpers tab.
2. Click Create Helper and pick Timer.
3. Name it something distinctive, for example `FP2 Living Room Heartbeat`. The entity id will be derived from the name, in this case `timer.fp2_living_room_heartbeat`.
4. Leave the duration field blank. The blueprint sets the duration dynamically every time it resets the timer; whatever you fill in here will be overwritten on the first report.
5. Save.

One timer helper per blueprint instance. If you watch three FP2s, you have three timers.

## Step 3: Create the Automation from the Blueprint

1. Go to Settings, then Automations & Scenes.
2. Open the Blueprints tab, find Sensor Watchdog, and click Create Automation.
3. Fill in the inputs:
    - Monitored Entities: the entities you want to watch. For an FP2 you can pick more than one (the illuminance sensor and the presence sensor, for example); reports from any of them count as a heartbeat.
    - Recovery Switch: the smart plug that powers the upstream device.
    - Heartbeat Timer: the timer helper you created in Step 2. Leave blank only if Detection Mode is set to minimal.
    - Detection Mode: responsive, lighter, or minimal. See the note below for guidance.
    - The remaining inputs have sensible defaults. Adjust the active window, scheduled reboot, and restart guard if you need them.
4. Save.

The automation is now armed. Watch it in Settings, Automations & Scenes, the Traces view, the first time a monitored entity reports. You should see a trace that resets the heartbeat timer and exits cleanly.

## A Small Note on the Three Detection Modes

The cost of catching a frozen device is event traffic. Home Assistant fires a `state_reported` event every single time any device sends a report, whether or not the value changed. In responsive mode, this blueprint's freshness detector wakes up for every one of those events system-wide and template-filters them down to the monitored entities. On a busy Home Assistant with hundreds of devices reporting every few seconds, that is a lot of trigger evaluations to ignore.

Lighter mode skips the `state_reported` event entirely and only listens for state *changes* on the monitored entities. The trigger is filtered at the trigger level by Home Assistant's normal entity_id matching, so the automation only wakes when one of the monitored entities actually changes value. This is the right mode for devices whose values move around enough that a quiet hour reliably means "device is broken." A presence sensor in an active room qualifies; an outdoor temperature sensor that holds the same value for ten minutes at a time does not.

Minimal mode is for the case where you do not need freshness detection at all. The device either drops offline (which the always-on unavailable detector catches) or wedges silently (which the optional scheduled cycle catches once a day at a quiet hour). No timer helper needed, no `state_reported` fan-out, no false positives from quiet rooms. The cost is that a silent freeze goes undetected until the scheduled cycle fires, which can be up to 24 hours.

If you do not know which mode fits, start with lighter. It is light enough to run on every device you have, and it catches the vast majority of freeze cases since most flaky devices either drop offline or stop reporting changes well before they stop sending reports entirely.

## A Small Note on the Active Window

The active window only gates the freshness detector, not the unavailable detector or the scheduled cycle. The reasoning is that the three detectors answer different questions and have different false-positive profiles.

The freshness detector asks "should this entity have reported by now," and the answer depends on whether the room is active. An FP2 in an empty bedroom at 3am has nothing to detect; not reporting is correct. Triggering a reboot in that case is a false positive. The window lets you say "only judge freshness during waking hours."

The unavailable detector asks "is this entity reachable at all," and the answer does not depend on the time of day. An FP2 that dropped offline at 3am is broken; it does not become unbroken at 6am just because the window opens. Gating the unavailable branch by a window would mean letting a broken device stay broken until morning, which is the opposite of what you want.

The scheduled cycle is preventive maintenance you have deliberately scheduled at a quiet hour for the explicit purpose of doing recovery during that hour. Gating it by a window would defeat its only purpose.

For v1.0.0 the active window cannot cross midnight. Set the start earlier than the end and you are fine. If you need a window that wraps midnight (for an environment that is "active" through the night), that is on the v1.1.0 list.

## A Small Note on One Attempt Per Freeze

When the freshness detector fires, the blueprint cycles the recovery switch and exits. It does not restart the heartbeat timer. That means a single freeze produces a single recovery attempt; if the device is still silent after the cycle completes, no further freshness-driven attempts will fire until the device starts reporting again and the timer resets normally.

The reason is that an infinite retry loop on a hard-failed device is worse than not retrying. A power supply that has died will not recover from a power cycle, but the blueprint cannot tell the difference between "device is wedged and a cycle will fix it" and "device is dead and a cycle will not." If it kept retrying, it would cycle the plug every freshness threshold forever, which serves no one.

The unavailable detector and the optional scheduled cycle are the safety nets here. If the device eventually drops fully offline, the unavailable detector will catch it (and that detector does fire on every transition, not once per freeze). If you want a guaranteed daily try-anyway, the scheduled cycle is what you turn on for that.

## A Small Note on the Self-Recovery for: Delay

The blueprint includes a small fact about its own behavior that is worth flagging: when the recovery cycle turns the switch off, that is itself a state change on the switch, and that state change would, naively, fire the self-recovery branch. The branch would then turn the switch back on a fraction of a second after the cycle just turned it off, defeating the cycle.

The blueprint avoids this with a `for:` delay on the self-recovery trigger: the switch must stay off for `recovery_off_seconds + 30` before the self-recovery branch fires. The recovery cycle's own turn-on happens within `recovery_off_seconds`, which cancels the `for:` delay before it ever expires, so the cycle's own turn-off does not fight itself. An external turn-off (a dashboard click, a voice command, a misfire from another automation) lasts longer than `recovery_off_seconds + 30`, so the self-recovery branch fires after that delay and puts things right.

The practical effect: a manual or accidental turn-off of the recovery switch takes a few tens of seconds to be corrected, not instantaneously. For a router plug that has other devices behind it, that is a small but acceptable network gap. For a use case where the recovery switch is dedicated to a single device, the gap is invisible.

## Worked Example

The original use case: three Aqara FP2 presence sensors at The Panorama, each plugged into its own smart plug. The living room FP2 wedges about twice a week and a power cycle fixes it. Three instances of the blueprint, one per FP2, watching both the presence and illuminance entities and cycling the matching plug:

Living Room FP2:

- Monitored Entities: `binary_sensor.fp2_living_room_presence`, `sensor.fp2_living_room_illuminance`
- Recovery Switch: `switch.fp2_living_room_plug`
- Recovery Off Duration: 10 seconds
- Unavailable Timeout: 30 seconds
- Detection Mode: Lighter
- Stale Threshold: 30 minutes
- Heartbeat Timer: `timer.fp2_living_room_heartbeat`
- Restrict Staleness to Active Window: enabled
- Active Window Start: 06:00:00
- Active Window End: 23:00:00
- Enable Scheduled Reboot: enabled
- Scheduled Reboot Time: 04:30:00
- Restart Guard Sensor: `sensor.restarting_status`

The living room is active enough during the day that 30 minutes without a value change reliably means the FP2 has wedged, and quiet enough at night that the active window prevents false positives. The scheduled cycle at 04:30 catches drift the freshness detector misses, and the restart guard keeps the blueprint quiet during the 03:00-04:00 reboot window. The same configuration with the entities and timer swapped applies to the master and guest FP2s.

Downstream automations that previously had to defend themselves against an `unavailable` FP2 (with `availability:` templates, `default_value` filters, or wrapping template sensors) can be left in place; the FP2 itself recovers within minutes of any wedge, and the unavailable window becomes short enough that downstream consumers rarely notice.

For more worked examples, including the router plug and WiFi camera scenarios, see the article: <https://xeazy.com/sensor-watchdog-blueprint/>

## Parameters

| Input | Required | Default | Accepted values | What it does |
| --- | --- | --- | --- | --- |
| `monitored_entities` | Yes | none | one or more entities | The entities watched by the blueprint. Reports from any of them reset the heartbeat timer; an unavailable transition on any of them triggers the unavailable branch. Typically multiple entities from the same physical device. |
| `recovery_switch` | Yes | none | a `switch` entity | The smart plug or other switch that powers the upstream device. Cycled (off, wait, on) by every recovery branch. |
| `recovery_off_seconds` | No | 10 | 1 to 60 | How long to hold the recovery switch off between the off action and the on action. Long enough for the device to fully power down. |
| `unavailable_timeout_seconds` | No | 30 | 5 to 600 | How long an entity must stay `unavailable` or `unknown` before the unavailable branch fires. Long enough to ride out transient outages without missing real failures. |
| `detection_mode` | No | `responsive` | `responsive`, `lighter`, `minimal` | Which mechanism feeds the freshness detector. Responsive catches every report including heartbeats (heaviest load). Lighter catches value changes only (lighter, recommended starting point). Minimal disables freshness detection entirely. |
| `stale_threshold_minutes` | No | 5 | 1 to 1440 | How long without a report (responsive) or without a value change (lighter) before the device is declared stale. Ignored in minimal mode. |
| `heartbeat_timer` | No (Yes for responsive and lighter) | none | a `timer` entity | The timer helper that tracks heartbeats. Create one timer per blueprint instance, with a blank duration. Required for responsive and lighter modes; ignored in minimal mode. |
| `active_window_enabled` | No | `false` | `true`, `false` | When enabled, the freshness branch only fires within the active time window below. Unavailable detection and scheduled reboot are not affected. |
| `active_after` | No | `06:00:00` | a time (HH:MM:SS) | Start of the active window. Ignored when the window is disabled. |
| `active_before` | No | `23:00:00` | a time (HH:MM:SS) | End of the active window. Must be later than `active_after`. v1.0.0 does not support windows that cross midnight. Ignored when the window is disabled. |
| `scheduled_reboot_enabled` | No | `false` | `true`, `false` | When enabled, the recovery cycle fires once per day at the scheduled time. Works in all detection modes including minimal. |
| `scheduled_reboot_time` | No | `04:30:00` | a time (HH:MM:SS) | The time of day to fire the scheduled cycle. Ignored when the scheduled cycle is disabled. |
| `restart_guard_sensor` | No | blank | any entity | When set, all recovery branches skip while this entity reads `on`, `true`, or `True`. Used to suppress the blueprint during a known reboot window. |
| `debug_enabled` | No | `false` | `true`, `false` | When enabled, the automation writes a log line for each branch it enters. Turn on during setup, turn off in normal operation. |

## Beta Status

This is 1.0.0-beta. The blueprint design has been simulated against 21 scenarios covering the obvious failure modes and a few non-obvious ones (the recovery cycle's own turn-off accidentally firing self-recovery, two monitored entities going unavailable in the same instant queueing duplicate cycles, the staleness branch firing during a reboot window). The author is running it against three Aqara FP2 sensors at home and will graduate it to 1.0.0 stable when that deployment has run clean for a sustained window with no false positives and no missed wedges.

**Verified in simulation:**

- All three detection modes route correctly through the choose: block.
- The active window gates only the freshness branch; the unavailable detector and the scheduled cycle ignore it.
- The restart guard blocks all three recovery branches.
- The self-recovery `for:` delay is long enough that the recovery cycle's own turn-off does not fire it.
- Simultaneous triggers (two entities unavailable in the same instant; stale and unavailable firing together) produce exactly one recovery cycle.
- The cycle is one-attempt-per-freeze; the timer is not auto-restarted after the staleness branch fires.

**Verified in production:** pending. The Panorama deployment starts on the living room FP2 in lighter mode with the active window enabled.

**Not yet verified:**

- The behavior in responsive mode on a busy installation. Responsive mode fires the freshness trigger for every `state_reported` event in Home Assistant, filtered down to the monitored entities in the action's first condition. The cost is documented but has not been measured on a real install.
- Behavior at Home Assistant startup when a monitored entity is already `unavailable` before HA finishes loading. State triggers fire on transitions, not initial states, so a wedge that predates the HA restart is detected only when the entity changes again or when the scheduled cycle fires.
- Multi-day scheduled cycles. v1.0.0 is daily-only; weekly or specific-days scheduling is on the v1.1.0 list.

If you run into any of the above before the author does, or you find a case the simulator missed, the forum thread is the right place to flag it.



Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>)

This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.

## Changelog

| Version | Notes |
| --- | --- |
| 1.0.0-beta | Initial public preview. Three independent failure detectors firing into a single recovery cycle: an unavailable detector with a tolerance window, a freshness detector backed by a timer helper, and an optional daily scheduled cycle. Three-mode selector (responsive, lighter, minimal) for the freshness detector's event load. Optional active time window for the freshness branch, optional restart guard input, and a self-recovery branch that uses a `for:` delay so the recovery cycle's own turn-off does not fight itself. Twenty-one simulator scenarios pass including simultaneous-trigger debouncing and the cycle-own-off case. Author-dogfooded only at release; will graduate to 1.0.0 stable after the home deployment runs clean. |
