# Sensor Watchdog (Beta)

A Home Assistant automation blueprint that watches one or more entities sharing a single smart plug and recovers the device behind that plug when those entities go offline or stop reporting fresh values. The blueprint watches the entities collectively: multiple entities from the same physical device often have complementary activity patterns, and pairing them gives better heartbeat coverage than either alone. A timer helper resets on any heartbeat from any monitored entity, so a device that freezes silently is caught, not just one that drops fully offline. An active time window prevents quiet empty rooms from being mistaken for frozen devices, and an optional daily scheduled cycle handles devices that benefit from a periodic preventive reboot.

An Aqara FP2 presence sensor talks over WiFi and roughly twice a week it locks up and stops reporting. Sometimes its entity drops to `unavailable`; sometimes the value just freezes wherever it happened to be, and Home Assistant has no way to know the device is dead until the next state change that never comes. A reboot of its smart plug fixes it every time. A Zigbee router plug at the back of the property occasionally drops its routing table and stops forwarding for the devices behind it; cycling its power restores the mesh within a minute. A WiFi camera with a known firmware bug needs a kick every few days. In each case the fix is the same trivial thing (turn the upstream plug off, wait ten seconds, turn it back on) and the only question is when to trigger it.

This blueprint answers that question three ways at once. It cycles the plug when any monitored entity goes `unavailable` for longer than a tolerance window. It cycles the plug when the entities collectively stop reporting fresh values for longer than a freshness threshold, which catches the freeze case the unavailable detector misses. And it can fire a daily preventive cycle on a schedule, for devices known from experience to need a periodic kick regardless of what their entities say. The detection is event-driven end to end: no time pattern polling, no scanning loops. A heartbeat timer is reset by reports from any of the monitored entities, and the recovery cycle fires when the timer expires.

Full write-up and the longer story behind the design: <https://xeazy.com/sensor-watchdog-blueprint/>  Questions and discussion: <https://xeazy.com/logbook/>

**Current version: 1.0.0-beta.** This blueprint is shipped as a public preview. The author is dogfooding it on three Aqara FP2 sensors at home and will not graduate it to 1.0.0 stable until that deployment has run clean for a sustained window. Use it on your own setup if you are comfortable troubleshooting; feedback on the forum is welcome.

## What You Get

- Three independent failure detectors firing into the same recovery cycle: an `unavailable` detector with a tolerance window, a freshness detector backed by a timer helper, and an optional daily scheduled cycle. An optional input also extends the unavailable detector to treat the `off` state as offline, so the blueprint can recover devices monitored via HA's ping integration (which reports `off`, not `unavailable`, when a target is unreachable).
- A three-way mode selector that lets you choose how much event traffic the freshness detector creates: responsive (catches heartbeats, heaviest load), lighter (catches state changes only, much lighter), or minimal (skip the freshness detector entirely and rely on the unavailable detector plus the optional scheduled cycle).
- An optional active time window for the freshness detector, so a presence sensor in an empty room overnight is not flagged as frozen just because nothing in the room is changing. The unavailable detector and the scheduled cycle ignore the window; a sensor that has dropped offline is broken at 3am the same as it is at 3pm.
- An optional restart guard input. Point it at a sensor that goes truthy during your Home Assistant or router reboot window, and all three recovery branches will skip while it is active. Stops the blueprint from cycling plugs in response to the natural unavailability that a reboot creates.
- A self-recovery branch that turns the recovery switch back on if it is found off for longer than the recovery cycle would normally hold it. Protects against a dashboard click or a voice command that accidentally takes the upstream plug offline.
- Debug logging that you can turn on during setup and turn off in normal operation. Log lines are written to the system log at info level, prefixed with `[Sensor Watchdog v<version>]` so they are easy to grep.

- Optional failure notification: when enabled, the blueprint waits a configurable settle period after each recovery cycle, then checks whether the monitored entities are still in a failure state. If they are, a user-configured action sequence runs - a notify call, a persistent_notification.create, a script call, anything you can express as a Home Assistant action. Disabled by default; opt-in.

A monitored entity counts as "online" whenever it has any state value at all that is not `unavailable` or `unknown`. The freshness detector measures from the entity's `last_reported` timestamp in responsive mode (which updates on every report including same-value heartbeats) or from `last_changed` in lighter mode (which only advances when the value actually changes).

## Where This Fits

The closest hand-rolled alternative is a one-off `Unavailable - <device>` automation written separately for each flaky device. That works for the offline case and is straightforward to write, but it misses freezes (the entity is technically still reporting, just frozen at the last value) and it duplicates the same hundred lines of YAML once per device. Three flaky FP2s mean three near-identical automations to maintain. Move the same logic into a blueprint and the YAML lives once, with one instance per device.

Polling-based watchdogs (a `time_pattern` that fires every five minutes and checks freshness in a template condition) catch the freeze case but at the cost of running an automation every five minutes forever, on every Home Assistant install, regardless of whether anything is wrong. The event-driven design here only does work when an actual report comes in or an actual timer expires.

This blueprint sits where you have one or more devices that wedge often enough to matter, where a power cycle fixes them, and where you want the recovery to be hands-off and visible in the automation traces. If you only have one such device and you are comfortable maintaining a bespoke automation for it, the bespoke automation may be the lighter answer. Past two or three, the blueprint pays for itself.

## Requirements

Home Assistant 2026.6 or newer. That is the version the blueprint is verified on.

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

## A Small Note on Pairing Entities

The blueprint watches one or more entities together against a single smart plug. One entity per instance works, but pairing two or more entities from the same physical device often produces better detection than either alone, because their activity patterns complement each other.

An Aqara FP2 exposes both a presence entity and an illuminance entity. The illuminance value changes constantly during the day as light levels shift; presence changes mostly in the evening when people are moving through the room under steady artificial light. The two entities take turns generating heartbeats: illuminance carries the daytime hours, presence carries the evening, and there is rarely a freshness window where neither is reporting. Watching only the presence entity would risk false freeze flags during long quiet stretches in the day. Watching only the illuminance entity would risk false flags in the evening when light levels are steady. Together they cover both halves of the day.

The pairing rule is straightforward: the entities all have to live on the same smart plug, since the recovery action is one plug cycle for the whole group. If you have two flaky devices on two different plugs, that is two instances of the blueprint, not one. If you have one device with three useful entities, that is one instance watching all three.

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

## A Small Note on What Counts as Offline

The offline detector watches for transitions to two specific states by default: `unavailable` and `unknown`. These are Home Assistant's standard markers for "I cannot reach this device" or "the integration has no value for this entity." A monitored entity moving into either state for longer than the offline timeout fires the recovery cycle.

Some integrations do not use those states. The clearest example is the built-in `ping` integration: when the target host is unreachable, a `ping` binary_sensor reports `off`, not `unavailable`. By default the blueprint does not treat `off` as a failure, because the vast majority of binary_sensor entities (presence detectors, door contacts, motion sensors, etc.) go to `off` as part of normal operation. Treating every `off` transition as a failure would cycle the recovery switch every time someone left a room or closed a door.

For ping-style monitoring, enable the `treat_off_as_offline` input. It adds a second offline trigger watching for `off` transitions on the monitored entities, with the same offline timeout. When the router stops responding to pings the binary_sensor goes `off`, the timeout completes, the plug cycles, and the router comes back online. See the WiFi router worked example below for a complete configuration.

**The setting is per-instance, not per-entity.** Every monitored entity in a blueprint instance shares the same `treat_off_as_offline` value. That means you cannot mix ping-style entities and normal binary_sensors in one instance: a presence sensor and a ping sensor on the same instance with `treat_off_as_offline` enabled would cycle the recovery switch every time the room emptied. The right approach for users with both types of devices is one instance per device class: one watchdog instance per FP2 (with `treat_off_as_offline` disabled), one watchdog instance per ping-monitored router (with `treat_off_as_offline` enabled). They can share a HA install but never an instance.

This split is also natural because the FP2 and the router have different recovery switches anyway. The pairing rule from earlier ("entities all on the same plug") already pushes you toward one instance per physical device, and the offline-state setting is one more reason to keep instances narrow.

## A Small Note on One Attempt Per Wedge

When any recovery branch fires (unavailable, stale, or scheduled), the blueprint cycles the recovery switch and then cancels the heartbeat timer. That cancellation matters: without it, an entity that goes unavailable would fire the unavailable branch, the cycle would run, and then a few minutes later the freshness detector would fire a second cycle for the *same* wedge because the timer (started at the last heartbeat before the entity went silent) would still be ticking. Cancelling the timer at the moment of recovery prevents the two detectors from stepping on each other.

The result is "one attempt per wedge" regardless of which detector caught it. After the cycle completes, the heartbeat timer is idle. If the recovered device starts reporting again the next heartbeat starts a fresh timer with the full stale threshold. If the device is still wedged after the cycle and never reports, the timer stays idle, no further freshness-driven attempts will fire, and the user has to wait for the unavailable trigger to re-fire on a new transition or for the scheduled cycle to take another preventive swing.

The reason for one-attempt-per-wedge rather than infinite retry is that a power supply that has died will not recover from a power cycle, but the blueprint cannot tell the difference between "device is wedged and a cycle will fix it" and "device is dead and a cycle will not." If it kept retrying, it would cycle the plug every stale threshold forever, which serves no one.

If a wedge proves consistently stubborn for a particular device (one cycle doesn't reliably fix it), the right answer is the optional scheduled cycle as a second daily try, or a shorter scheduled interval, not auto-retry within a single wedge.

## A Small Note on the Self-Recovery for: Delay

The blueprint includes a small fact about its own behavior that is worth flagging: when the recovery cycle turns the switch off, that is itself a state change on the switch, and that state change would, naively, fire the self-recovery branch. The branch would then turn the switch back on a fraction of a second after the cycle just turned it off, defeating the cycle.

The blueprint avoids this with a `for:` delay on the self-recovery trigger: the switch must stay off for `recovery_off_seconds + 30` before the self-recovery branch fires. The recovery cycle's own turn-on happens within `recovery_off_seconds`, which cancels the `for:` delay before it ever expires, so the cycle's own turn-off does not fight itself. An external turn-off (a dashboard click, a voice command, a misfire from another automation) lasts longer than `recovery_off_seconds + 30`, so the self-recovery branch fires after that delay and puts things right.

The practical effect: a manual or accidental turn-off of the recovery switch takes a few tens of seconds to be corrected, not instantaneously. For a router plug that has other devices behind it, that is a small but acceptable network gap. For a use case where the recovery switch is dedicated to a single device, the gap is invisible.

## A Small Note on Failure Notification

A recovery cycle is a best effort. The blueprint cycles the upstream plug, the device gets a fresh boot, and most of the time the entities come back online and life goes on. Sometimes the device is dead, the power supply is gone, the WiFi credentials have aged out, the SD card has corrupted, and a power cycle will not save it. Without something else doing the watching, those failures sit there quietly until you notice the room is still dark or the data is still wrong.

The Failure Notification feature is the something else. When you enable it, every recovery cycle takes a settle period after the switch turns back on, gives the device a chance to boot and reconnect, then looks at the monitored entities and asks whether any are still in a failure state. If everything is healthy the cycle exits silently and you would never know it ran. If anything is still broken the configured On Failure Action runs.

The settle period (default 60 seconds) is the only tunable that really matters. Set it to roughly the slowest boot time of the devices you watch: 30-60 seconds for most Zigbee plugs and sensors, 60-90 for WiFi routers, 90-120 for NAS units or anything else that takes a real moment to come up. Setting it too short means you get false failure notifications when the device is just still booting. Setting it too long is fine for correctness; you just learn about failures a bit later.

The On Failure Action input takes a full Home Assistant action sequence: notify calls, persistent_notification.create, script calls, MQTT publish, anything. Two variables are available inside that sequence: `recovery_reason` (one of `unavailable`, `stale`, `scheduled`) and `failed_entities` (a list of monitored_entities still in a failure state). A typical setup notifies a phone or chat channel; a thorough one also creates a persistent notification in Home Assistant so the failure is visible on the dashboard until acknowledged. See the worked examples below.

The Self-Recovery branch does not run failure detection: it is just turning the switch back on after an external off, not power-cycling a device that needs to come back, so there is no "success or failure" to evaluate.

## Worked Examples

### FP2 presence sensors

The original use case: three Aqara FP2 presence sensors at The Panorama, each plugged into its own smart plug. The living room FP2 wedges about twice a week and a power cycle fixes it. Three instances of the blueprint, one per FP2, watching both the presence and illuminance entities and cycling the matching plug:

Living Room FP2:

- Monitored Entities: `binary_sensor.fp2_living_room_presence`, `sensor.fp2_living_room_illuminance`
- Recovery Switch: `switch.fp2_living_room_plug`
- Recovery Off Duration: 10 seconds
- Offline Timeout: 30 seconds
- Detection Mode: Lighter
- Stale Threshold: 30 minutes
- Heartbeat Timer: `timer.fp2_living_room_heartbeat`
- Restrict Staleness to Active Window: enabled
- Active Window Start: 06:00:00
- Active Window End: 23:00:00
- Enable Scheduled Reboot: enabled
- Scheduled Reboot Time: 04:30:00
- Treat 'off' as Offline: **disabled** (the presence entity legitimately goes `off` when the room is empty)
- Restart Guard Sensor: `sensor.restarting_status`

The living room is active enough during the day that 30 minutes without a value change reliably means the FP2 has wedged, and quiet enough at night that the active window prevents false positives. The scheduled cycle at 04:30 catches any silent wedge the freshness detector misses, and the restart guard keeps the blueprint quiet during the 03:00-04:00 reboot window. The same configuration with the entities and timer swapped applies to the master and guest FP2s.

`Treat 'off' as Offline` is left off because the presence entity going `off` is a normal event (the room emptied), not a failure. The offline detector catches the FP2's failure modes (`unavailable` and `unknown` states) without needing to treat `off` as a problem.

Downstream automations that previously had to defend themselves against an `unavailable` FP2 (with `availability:` templates, `default_value` filters, or wrapping template sensors) can be left in place; the FP2 itself recovers within minutes of any wedge, and the unavailable window becomes short enough that downstream consumers rarely notice.

### WiFi router with a ping sensor

A second use case from The Panorama: a flaky WiFi router that occasionally locks up and needs a power cycle to come back. Monitored via HA's built-in `ping` integration; recovered by the Zigbee smart plug it lives on. This goes in a **separate** blueprint instance from any FP2 or normal-binary-sensor instances, because `Treat 'off' as Offline` is a per-instance setting that applies to every monitored entity in that instance.

First, configure HA's ping integration in `configuration.yaml`:

```yaml
binary_sensor:
  - platform: ping
    host: 192.168.1.1
    name: router_gateway_ping
    count: 3
    scan_interval: 30
```

This produces `binary_sensor.router_gateway_ping`, which reports `on` when the gateway is reachable and `off` when it is not. Then create the watchdog instance from the blueprint:

- Monitored Entities: `binary_sensor.router_gateway_ping`
- Recovery Switch: `switch.wifi_router_zigbee_plug`
- Recovery Off Duration: 15 seconds
- Offline Timeout: 90 seconds
- Detection Mode: **Minimal** (the ping sensor sits at `on` for days when the router is healthy; freshness detection in `responsive` or `lighter` mode would falsely flag the long-quiet stretches)
- Heartbeat Timer: (blank; minimal mode does not need one)
- Treat 'off' as Offline: **enabled** (the whole reason we are using a ping sensor)
- Enable Scheduled Reboot: enabled
- Scheduled Reboot Time: 04:00:00
- Restart Guard Sensor: `sensor.restarting_status`

In minimal mode the offline detector is the workhorse. The 90-second offline timeout rides out brief network blips and short DHCP renewals without falsely cycling the router. When the gateway has been unreachable for more than 90 seconds the plug cycles, the router boots, and the gateway comes back online within another minute or so. The 04:00 scheduled cycle is preventive maintenance, useful for routers with slow memory leaks.

The Zigbee plug for the recovery switch is deliberate. If the recovery switch were a WiFi smart plug, it would also be offline during the outage and the cycle could not run. A Zigbee plug stays reachable through the Zigbee coordinator regardless of what the WiFi network is doing.

For the router we also want to know if the cycle did NOT bring it back, since a dead router means the internet is down indefinitely. Configure the Failure Notification section:

- Enable Failure Notification: **enabled**
- Recovery Settle Time: 120 seconds (a router needs about that long to boot and re-establish its WAN link)
- On Failure Action:

```yaml
- service: notify.mobile_app_james_phone
  data:
    title: "Router watchdog: cycle did not recover"
    message: >-
      The router was cycled at {{ now().strftime('%H:%M') }} because
      {{ recovery_reason }}, but {{ failed_entities | join(', ') }}
      is still failed after the settle period.
- service: persistent_notification.create
  data:
    title: "Router watchdog failure"
    message: >-
      Cycle ran (reason: {{ recovery_reason }}). Entities still failed:
      {{ failed_entities | join(', ') }}. Manual intervention may be
      required.
    notification_id: "sensor_watchdog_router_failure"
```

Two actions, in sequence: a push notification to the phone, plus a persistent notification in the Home Assistant dashboard so the failure does not get missed if the phone is on silent. The persistent notification has a fixed `notification_id`, so repeated failures update the same notification rather than stacking dozens of duplicates. The `recovery_reason` and `failed_entities` variables are injected automatically by the blueprint.

For more worked examples, including Zigbee router plugs, smart cameras, and NAS scenarios, see the article: <https://xeazy.com/sensor-watchdog-blueprint/>

## Parameters

| Input | Required | Default | Accepted values | What it does |
| --- | --- | --- | --- | --- |
| `monitored_entities` | Yes | none | one or more entities | The entities watched by the blueprint. All must live on the same recovery_switch (one plug cycle is the recovery action for the whole group). Reports from any of them reset the heartbeat timer; an unavailable transition on any of them triggers the unavailable branch. Pairing multiple entities from the same physical device often improves detection because their activity patterns complement each other; see "A Small Note on Pairing Entities" above. |
| `recovery_switch` | Yes | none | a `switch` entity | The smart plug or other switch that powers the upstream device. Cycled (off, wait, on) by every recovery branch. |
| `recovery_off_seconds` | No | 10 | 1 to 60 | How long to hold the recovery switch off between the off action and the on action. Long enough for the device to fully power down. |
| `unavailable_timeout_seconds` | No | 30 | 5 to 600 | How long an entity must stay in an offline state before recovery fires. "Offline" means `unavailable` or `unknown` by default; with `treat_off_as_offline` enabled, also includes `off`. Long enough to ride out transient outages without missing real failures. |
| `detection_mode` | No | `responsive` | `responsive`, `lighter`, `minimal` | Which mechanism feeds the freshness detector. Responsive catches every report including heartbeats (heaviest load). Lighter catches value changes only (lighter, recommended starting point). Minimal disables freshness detection entirely. |
| `stale_threshold_minutes` | No | 5 | 1 to 1440 | How long without a report (responsive) or without a value change (lighter) before the device is declared stale. Ignored in minimal mode. |
| `heartbeat_timer` | No (Yes for responsive and lighter) | none | a `timer` entity | The timer helper that tracks heartbeats. Create one timer per blueprint instance, with a blank duration. Required for responsive and lighter modes; ignored in minimal mode. |
| `active_window_enabled` | No | `false` | `true`, `false` | When enabled, the freshness branch only fires within the active time window below. Unavailable detection and scheduled reboot are not affected. |
| `active_after` | No | `06:00:00` | a time (HH:MM:SS) | Start of the active window. Ignored when the window is disabled. |
| `active_before` | No | `23:00:00` | a time (HH:MM:SS) | End of the active window. Must be later than `active_after`. v1.0.0 does not support windows that cross midnight. Ignored when the window is disabled. |
| `scheduled_reboot_enabled` | No | `false` | `true`, `false` | When enabled, the recovery cycle fires once per day at the scheduled time. Works in all detection modes including minimal. |
| `scheduled_reboot_time` | No | `04:30:00` | a time (HH:MM:SS) | The time of day to fire the scheduled cycle. Ignored when the scheduled cycle is disabled. |
| `treat_off_as_offline` | No | `false` | `true`, `false` | When enabled, a monitored entity transitioning to `off` is treated the same as one transitioning to `unavailable` or `unknown` and triggers recovery after the offline timeout. Enable only for entities whose `off` state means failure (e.g., binary_sensors from HA's `ping` integration, which report `off` when the target is unreachable). Leave disabled for entities whose `off` state is normal operation (presence sensors, motion sensors, door contacts, switches). The setting is per-instance: every monitored entity in this instance shares the same value. See "A Small Note on What Counts as Offline" above. |
| `failure_notification_enabled` | No | `false` | `true`, `false` | When enabled, every recovery cycle is followed by a settle delay and a check for whether any monitored entity is still in a failure state. If yes, the On Failure Action runs. Disabled by default; opt-in. See "A Small Note on Failure Notification" above. |
| `recovery_settle_seconds` | No | 60 | 5 to 600 | How long to wait after the recovery switch turns back on before checking whether the device recovered. Roughly the boot time of the slowest device on the recovery switch. Ignored when failure notification is disabled. |
| `on_failure_action` | No | empty list | a Home Assistant action sequence | The action sequence to run when a recovery cycle fails to bring the device back online. Typically a `notify` call, a `persistent_notification.create`, a script call, or any combination. Two variables are available inside the sequence: `recovery_reason` (one of `unavailable`, `stale`, `scheduled`) and `failed_entities` (a list of monitored_entities still in a failure state). |
| `restart_guard_sensor` | No | blank | any entity | When set, all recovery branches skip while this entity reads `on`, `true`, or `True`. Used to suppress the blueprint during a known reboot window. |
| `debug_enabled` | No | `false` | `true`, `false` | When enabled, the automation writes a log line for each branch it enters. Turn on during setup, turn off in normal operation. |

## Beta Status

This is 1.0.0-beta. The blueprint design has been simulated against 43 scenarios covering the obvious failure modes and a number of non-obvious ones (the recovery cycle's own turn-off accidentally firing self-recovery, two monitored entities going unavailable in the same instant queueing duplicate cycles, the staleness branch firing during a reboot window, the unavailable and stale detectors firing in sequence for the same wedge, the failure notification firing on persistent failure and not firing on successful recovery). The author is running it against three Aqara FP2 sensors at home and will graduate it to 1.0.0 stable when that deployment has run clean for a sustained window with no false positives and no missed wedges.

**Verified in simulation:**

- All three detection modes route correctly through the choose: block.
- The active window gates only the freshness branch; the unavailable detector and the scheduled cycle ignore it.
- The restart guard blocks all three recovery branches.
- The self-recovery `for:` delay is long enough that the recovery cycle's own turn-off does not fire it.
- Simultaneous triggers (two entities unavailable in the same instant; stale and unavailable firing together) produce exactly one recovery cycle.
- The cycle is one-attempt-per-wedge across all three detectors: the unavailable and scheduled recovery branches cancel the heartbeat timer at the moment of cycle, so the freshness detector does not fire a second cycle for the same wedge.
- The `treat_off_as_offline` input correctly extends the offline detector to `off` transitions when enabled, and leaves the default (unavailable/unknown only) behaviour unchanged when disabled.
- Failure notification fires when the cycle does not recover the device by the end of the settle window, does NOT fire when the device recovers in time, and is correctly disabled by default. The self-recovery branch does not run failure notification.

**Verified in production:** pending. The Panorama deployment starts on the living room FP2 in lighter mode with the active window enabled.

**Not yet verified:**

- The behavior in responsive mode on a busy installation. Responsive mode fires the freshness trigger for every `state_reported` event in Home Assistant, filtered down to the monitored entities in the action's first condition. The cost is documented but has not been measured on a real install.
- Behavior at Home Assistant startup when a monitored entity is already `unavailable` before HA finishes loading. State triggers fire on transitions, not initial states, so a wedge that predates the HA restart is detected only when the entity changes again or when the scheduled cycle fires.
- Multi-day scheduled cycles. v1.0.0 is daily-only; weekly or specific-days scheduling is on the v1.1.0 list.

If you run into any of the above before the author does, or you find a case the simulator missed, the forum thread is the right place to flag it.

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>)

This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.

## Changelog

| Version | Notes |
| --- | --- |
| 1.0.0-beta | Initial public preview. Three independent failure detectors firing into a single recovery cycle: an unavailable detector with a tolerance window, a freshness detector backed by a timer helper, and an optional daily scheduled cycle. Three-mode selector (responsive, lighter, minimal) for the freshness detector's event load. Optional active time window for the freshness branch, optional restart guard input, optional `treat_off_as_offline` input to extend the offline detector to `off` transitions (for ping-style sensors), and a self-recovery branch that uses a `for:` delay so the recovery cycle's own turn-off does not fight itself. Unavailable and scheduled recovery branches cancel the heartbeat timer at cycle time, so the freshness detector does not fire a second cycle for the same wedge (one-attempt-per-wedge invariant). Optional failure notification: enable to wait a configurable settle period after each cycle, check whether monitored entities are still in a failure state, and run a user-configured action sequence (notify, persistent_notification, etc.) with `recovery_reason` and `failed_entities` variables available in the action context. Debug logging is written at info level with a `[Sensor Watchdog v<version>]` prefix, matching the Thinking Home blueprint style. Forty-three simulator scenarios pass including simultaneous-trigger debouncing, the cycle-own-off case, the FP2 10-zone case, the WiFi router ping scenario, the unavailable-into-stale double-fire case, and the failure notification firing-on-persistent-failure case. Author-dogfooded only at release; will graduate to 1.0.0 stable after the home deployment runs clean. |
