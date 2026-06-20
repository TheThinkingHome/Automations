# Linked Entities Pro (Beta)

A Home Assistant automation blueprint that keeps two or more entities in sync. When they briefly disagree, a chosen tiebreaker wins.

Picture two independent switches, a switch for the hallway lights and a switch for the stairway lights. Anyone walking through presses both. Now picture a wall switch wired to the lights, paired with a second virtual switch across the room. Or picture a smart bulb that stays powered all the time, controlled by another virtual wall switch that sends events instead of controlling the bulb directly. In each of these setups, the controllers can fall out of sync. When they do, the next press lands wrong or gets ignored, the dashboard shows one thing while the fixture does another, and the room is broken until someone notices. The naive two-way sync automation either ping-pongs in a loop, or carries a recursion guard so loose that it fires intermittently. *Linked Entities Pro* fixes both. A state-equality gate stops every run when the other linked entities match the source's new state, so the followers' responses cannot drive the loop any further. Any number of entities can sit in one linked group, in any mix of integrations, and a change on any of them propagates to the others.

**A note on scope.** This release syncs binary on/off state only. Brightness, color, and fan speed do not propagate, by design. Light and fan domains are supported because their on/off state still syncs cleanly; brightness/color/speed propagation is planned for v1.1.0.

Two events can leave the group out of sync. On Home Assistant restart, entities come back online in unpredictable order, some still carrying stale states from before. On a Zigbee2MQTT bridge reconnect, the bridge republishes every device's last-known state at once, which looks like every switch in the house just got pressed by an invisible hand. In both moments the linked entities can disagree, and the blueprint cannot tell from the events alone which state represents reality. That is where the *authority entity* comes in. You designate one entity in the group as the source of truth for these moments. When a reconcile event fires, the authority's current state is treated as correct and the others are commanded into alignment. During ordinary use the authority has no special role: any linked entity can drive the others. It only matters in those two narrow situations.

Full write-up and the longer story behind the design: <https://xeazy.com/linked-entity-pro/>  Questions and discussion: <https://xeazy.com/logbook/d/38-linked-entities-pro-blueprint>

## What You Get

- Two-way sync across two or more entities. A change on any of them propagates to the others.
- Cross-domain support. Switches, lights, input_booleans, fans, and groups can mix freely in the same linked group.
- A state-equality gate that stops a run when the other linked entities match the source's new state, so the followers' responses cannot loop the trigger back on itself.
- A reconcile branch that runs on Home Assistant start and on an optional bridge sensor reporting back online, with a designated authority entity acting as the tiebreaker.
- An optional peer-sync block for the first minute or two after Home Assistant starts, gated on the built-in Uptime integration. Prevents republished stale states from propagating during boot. The reconcile branches still run.
- Debug logging that can be toggled on during setup and off in normal operation.

Trigger filtering on the linked entities is restricted to `to: ['on', 'off']`, so devices coming back online from `unavailable` do not fire the trigger. The reconcile branch waits up to 30 seconds for the authority to enter a known state before acting, so an unavailable authority on boot does not corrupt the alignment.

## Where This Fits

There are a few common ways to keep entities in sync in Home Assistant, and each has its place.

A one-direction automation is the right tool when the dependency really is one-way. A motion sensor triggers a light, and the light has no reason to ever drive the sensor. This approach is cheap and obvious.

A pair of mirrored automations, one for A drives B and one for B drives A, covers the two-way case. They are easy to write and easy to break. Without a recursion guard you get a ping-pong loop. With a naive guard you get a loop that fires intermittently. With a strict guard you get drift on restart that nothing resolves. Most published switch-sync blueprints sit here.

Native Home Assistant Light Groups handle groups of any size when every member is in the `light` domain. The group acts as the controllable parent: turn the group on and every member turns on. But the sync only flows downward, from the group to its members. Change one member directly, from its own wall switch or its own card, and the siblings do not follow. Groups also do not give you cross-domain mixing (a wall switch and a relay are not lights), or alignment of the underlying entities after a bridge reconnect that republishes them out of order.

This blueprint sits where you want two-way sync across two or more entities, want it to survive restarts and bridge reconnects without drift, and want to mix domains (switches, lights, input_booleans, fans, groups) freely.

## Requirements

Home Assistant 2026.6.0 or newer. That is the version it is verified on.

Optional: the built-in HA Uptime integration, if you want the HA-start peer-sync block. Setup is one click via Settings > Devices & Services > Add Integration > Uptime. The blueprint works fine without it.

## Step 1: Import the Blueprint

The quick way, one click:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTheThinkingHome%2FAutomations%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Flinked_entities_pro.yaml)

Or by hand:

1. Go to Settings, then Automations & Scenes, then the Blueprints tab.
2. Click Import Blueprint at the bottom right.
3. Paste this URL and click Preview:

```
https://github.com/TheThinkingHome/Automations/blob/main/blueprints/automation/linked_entities_pro.yaml
```

4. Confirm and Import.

After import, the blueprint appears on the Blueprints page with a Create Automation button next to it.

## Step 2: Create the Automation

Click Create Automation on the imported blueprint. Home Assistant opens the automation editor with the blueprint's inputs grouped into four sections.

- **📦 Linked Entities**: pick the entities you want to keep in sync, and optionally designate one as the authority.
- **⏱ Sync Delay**: set the pause between trigger and dispatched actions.
- **🔄 Reconciliation**: turn the Home Assistant start reconcile on or off, optionally point at a bridge sensor (Zigbee2MQTT users want `binary_sensor.zigbee2mqtt_running`), and optionally enable the HA-startup peer-sync block via the Uptime integration.
- **🛠 Advanced**: toggle debug logging while you confirm the setup.

Give the automation a name, save it, and the sync is live.

The full parameter reference is in the table at the end of this page.

## A Small Note on Loop Prevention

A two-way sync automation can chase its own tail if it is not careful. When the blueprint commands a follower into alignment, that follower's state change fires the same trigger that started the work, and a naive design will keep going. *Linked Entities Pro Beta* breaks the loop with a state-equality check: before doing anything, the automation asks whether the other linked entities already match the source's new state. If they do, it stops. If they do not, it dispatches the commands. The followers respond, their state changes fire the trigger again, the gate checks again, and as soon as everyone agrees the work ends.

The reason this is enough comes from how Home Assistant handles state changes. `state_changed` only fires when the state actually changes. If the blueprint commands an already-on switch to turn on, the device reports back its unchanged state, and no event fires. So the gate only has to handle real propagation, never imaginary echoes.

This design has a useful property the strict context-filter approach does not: any state change propagates, regardless of who caused it. A physical press, a dashboard toggle, a scene automation, and a voice command all flow through the same gate and reach the followers the same way.

## A Small Note on Restart and Bridge Drift

Two events sit outside the peer sync model and have their own branch.

A Home Assistant restart brings entities back up in unpredictable order. Some report stale states from before the restart, some come up unavailable for a while and then settle, and the linked group can disagree for several seconds while everything finishes loading. The reconcile branch reads the authority entity's state, waits up to 30 seconds for it to enter a known on or off state, then commands every other entity in the group to match. If the authority is still unavailable after the 30-second wait, the reconcile exits cleanly and waits for the next event to try again.

A Zigbee2MQTT bridge reconnect is the second event. Z2M republishes every device's last-known state on reconnect, which arrives as a cascade of state-change events in unpredictable order. The peer sync branch reading those alone would race; whichever entity republishes last would drive everyone else to its state, regardless of which state is actually correct. The reconcile trigger sensor input solves this: point it at `binary_sensor.zigbee2mqtt_running`, and the reconcile branch runs on its off-to-on transition, treating the authority as truth and aligning everything else.

Users of other Zigbee integrations or other transports can usually leave the reconcile trigger sensor blank: ZHA does not republish on coordinator reconnect, Z-Wave JS tracks node state internally and does not flood events, and WiFi devices reconnect per device, not as a cascade. See the table further down.

## A Small Note on the HA Start Block Window

The reconcile branch handles drift after Home Assistant restarts. The HA start block window prevents a different problem: a republished stale state from one device propagating through peer sync to the others before the reconcile has caught up. The two work together.

Picture the kitchen wall switch and the Tasmota relay on the same fixture, both via Z2M. Home Assistant starts. The Z2M bridge takes 45 seconds to fully come up. As it does, the bridge republishes the wall switch's state (off, from before the restart) and the relay's state (on, also from before the restart). Without a block, the off state arrives first, peer sync fires, and the blueprint commands the relay off. Then the relay's actual state (on) arrives a moment later, peer sync fires again, and the blueprint commands the wall switch on. The fixture is now in the state it was before the restart, but it took two pointless dispatches to get there, and during that window the light was wrong.

The HA start block window stops that. When `use_uptime_sensor` is checked and uptime is below `ha_start_block_seconds`, the peer sync branch quietly skips. The reconcile branch, which is the intended response to startup drift, still runs. The authority entity wins, and the followers are aligned in one pass instead of a cascade.

Setup is straightforward. Settings > Devices & Services > Add Integration > Uptime, click through, you get `sensor.uptime`. The integration reports the moment Home Assistant last started as an ISO8601 timestamp (the only mode the modern config-flow Uptime integration produces). The blueprint reads that timestamp and computes elapsed seconds via `(now() - sensor_value).total_seconds()`. No configuration needed. If you do not want the block at all, leave `use_uptime_sensor` unchecked and the blueprint behaves as it did before the input existed.

The block applies only to peer sync. The reconcile-on-HA-start branch and the reconcile-trigger-sensor branch are not blocked, because they are the intended response to boot drift. Skipping them would defeat the point of having them.

## A Small Note on Cascade Timing

A sync run has three phases. First, the trigger fires and the loop-prevention gate decides whether to proceed. Second, the automation waits for `sync_delay_ms` before dispatching. Third, the dispatch fires and the followers report their new states some milliseconds later. Most of the time this is invisible; the cascade ends in well under a second. Two cases are worth knowing about.

**A second press during the delay** is handled cleanly. The automation runs in `mode: restart`, so the new trigger cancels the in-flight run before it dispatches anything. The new press takes over, the loop-prevention gate evaluates the new state, and the system settles to that. This is the behavior you want: press on, change your mind a moment later, press off, and the followers go off (or stay off). The first press never leaves the source.

**A second press *after* the dispatch but before the followers have finished responding** is the limitation. The dispatch is already out the door; the followers are turning on. When you press off in the middle of that, the source reports off, the loop-prevention gate looks around and sees the followers haven't responded yet, and the run quietly skips. Then the followers' responses arrive, each one looking like a fresh "turn on" event, and the cascade propagates back into the source. The net effect is that the second press is lost, and the group settles on the state the first press wanted. The system never gets stuck or loops; it just lands on the wrong side of the toggle.

In practice the window for this is small. With the default 200 ms sync delay and typical Zigbee response of 250-500 ms, the second-press window is roughly half a second between the dispatch and the slowest follower's reply. If you find yourself flipping the same entity in opposite directions inside that window often, raise `sync_delay_ms` to extend the safe cancellation window, or simply press again after the cascade settles. The reconcile branches catch any drift the next time Home Assistant restarts or the bridge reconnects.

## Worked Example

A common one: a fixture is controlled by two physical things, a Zigbee wall switch and a WiFi relay (a Tasmota module wired behind the cabinet, closer to the actual load). Either can turn the light on or off, and both have to stay in sync, because the next press of the wall switch will go the wrong way if the two report different states.

Configure the blueprint like this:

- **Linked Entities**: `switch.kitchen_counter_wall_switch`, `switch.kitchen_counter_relay`
- **Authority Entity**: `switch.kitchen_counter_relay` (closer to the load, so it carries the true state if the two disagree)
- **Reconcile on Home Assistant Start**: ON
- **Reconcile Trigger Sensor**: `binary_sensor.zigbee2mqtt_running`
- **Block Peer Sync During HA Startup**: ON, if you have the Uptime integration installed
- **Uptime Sensor**: `sensor.uptime`
- **Block Peer Sync After HA Start (seconds)**: 120
- **Sync Delay**: 0 (a two-entity pair has no mesh-load risk, so throttling is not needed)
- **Debug Logging**: ON for the first day, OFF after

Press the wall switch and the relay follows. Toggle the relay from the dashboard and the wall switch follows. Restart Home Assistant and the wall switch is pulled into alignment with the relay. Cycle the Z2M addon and the same thing happens. The two states never drift apart.

The startup block matters most for Z2M-heavy setups. When the bridge reconnects after a restart, it republishes every device's state in unpredictable order, and the block prevents the peer sync from chasing those republished states while the reconcile branch is doing its work. For non-Z2M setups, the block is less critical but still useful as a guard against any stale-state cascade.

For more worked examples, including mixed-integration groups and the longer story behind each design choice, see the article: <https://xeazy.com/linked-entity-pro/>

## Reconcile Trigger Sensor by Integration

| Integration | Recommended sensor | Why |
| --- | --- | --- |
| Zigbee2MQTT | `binary_sensor.zigbee2mqtt_running` | Z2M republishes every device's state on bridge reconnect, which arrives as a cascade of state changes in unpredictable order. This is the situation the input was built for. |
| ZHA | Usually leave blank | ZHA does not republish states on coordinator reconnect. The Home Assistant start reconcile is enough. |
| Z-Wave JS | Usually leave blank | The driver tracks node state internally and does not flood events on reconnect. |
| WiFi (Shelly, Tasmota, etc.) | Usually leave blank | Reconnects are per device, not cascaded. |
| Mixed | Pick the most failure-prone bridge | One sensor, used for the integration that causes the most drift. |

## Parameters

| Input | Required | Default | Accepted values | What it does |
| --- | --- | --- | --- | --- |
| `linked_entities` | Yes | none | two or more entities from `switch`, `light`, `input_boolean`, `fan`, or `group` | The entities to keep in sync. A change on any of them propagates to the others. |
| `authority_entity` | No | first entity in `linked_entities` | one entity from the same domain set | The tiebreaker entity. Used only on reconcile events, never during ordinary use. |
| `reconcile_on_ha_start` | No | `true` | `true`, `false` | When true, a reconcile runs on Home Assistant start. |
| `reconcile_trigger_sensor` | No | none | any `binary_sensor` | When set, a reconcile runs on the sensor's off-to-on transition. Catches bridge-reconnect cascades. |
| `use_uptime_sensor` | No | `false` | `true`, `false` | When true, peer sync is suppressed while HA uptime is below `ha_start_block_seconds`. Reconcile branches are not blocked. Requires the Uptime integration. |
| `uptime_sensor` | No | `sensor.uptime` | any `sensor` entity whose state parses as an ISO8601 timestamp via `as_datetime()` | Consulted only when `use_uptime_sensor` is true. The blueprint expects the modern HA Uptime integration's timestamp mode; elapsed seconds computed via `(now() - state).total_seconds()`. |
| `ha_start_block_seconds` | No | `120` | 0 to 900 | How long after HA starts to suppress peer sync. Set to 0 to disable the block while keeping the configuration. |
| `sync_delay_ms` | No | `200` | a whole number of milliseconds, 0 to 5000 | Milliseconds between the trigger and the dispatched commands. Doubles as the cancellation window for rapid opposite presses, so a larger value gives `mode: restart` more room to absorb in-flight changes. Setting it at or above the slowest device's response latency (typically 200 to 500 ms for Zigbee) eliminates redundant commands during the cascade. Set to 0 for a pair on fast hardware where there is nothing for the delay to absorb. |
| `debug_enabled` | No | `false` | `true`, `false` | When true, writes a log line for every trigger and action. Turn off in normal operation. |

A note on groups. If you add a group entity to the linked list along with one of its own members, the sync still works correctly. The state-equality gate prevents the infinite loop you would otherwise get. The cost is one or two redundant `homeassistant.turn_*` service calls per state change, since the group commands its member which is already being commanded directly. For cleanliness, prefer to link either the group or its members, not both.

## Beta Status

This is a public preview, currently at v1.0.3-beta. The blueprint runs in production at the original home it was built for, with two pairs verified: a Zigbee wall switch and a Tasmota relay (cross-integration), and two Z2M switches on the same fixture (same-integration). It has not yet been validated by an external tester on a different setup. Once an external tester confirms it works in their environment, the blueprint graduates from beta and the version becomes 1.0.0 stable. The file path stays at `blueprints/automation/linked_entities_pro.yaml` (the `_beta` suffix on the filename was dropped in v1.0.3-beta).

Want to be that tester? [Drop into the forum thread](https://xeazy.com/logbook/d/38-linked-entities-pro-blueprint) and say hi.

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>)

This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.

## Changelog

| Version | Notes |
| --- | --- |
| 1.0.3-beta | Three structural changes. (1) Gate template for the HA-start block window rewritten to handle timestamp-mode `sensor.uptime`. The earlier template did `states(uptime_sensor) | float(99999)` and a `unit_of_measurement` lookup, which worked only when the Uptime integration was in numeric duration mode. The modern config-flow Uptime integration in HA 2026.6+ produces only timestamp mode (an ISO8601 datetime like `2026-06-16T08:32:18+00:00`), where `float()` failed and fell back to the sentinel `99999`. The comparison `99999 < ha_start_block_seconds` was always false, so the gate was silently inert and the HA-start block never engaged. New template computes `(now() - as_datetime(states(uptime_sensor))).total_seconds()`. The reconcile branches and post-restart branches were unaffected because they use the input directly as a `delay:`, not through the gate. (2) Per-target availability filter on the peer-sync action. Before dispatching `homeassistant.turn_*`, the target list is filtered to exclude any linked entity currently in `unavailable` or `unknown` state. Stops HA from logging "Referenced entities X are missing or not currently available" service warnings when an upstream component (router, Z2M coordinator) restarts and entities flap to unavailable while HA itself remains up. Targets that come back will sync on their next state change. (3) Filename renamed from `linked_entities_pro_beta.yaml` to `linked_entities_pro.yaml`. The path no longer changes at graduation. `source_url` in the YAML and import links in this README updated accordingly. **Breaking**: users on the previous file path will need to re-import via the new URL on next blueprint refresh; existing imports continue to work without action. |
| 1.0.2-beta | Promoted `sync_delay_ms` to its own top-level input section, placed between Linked Entities and Reconciliation. The section makes the parameter visible and signals that it is a key tuning input that should be considered for every setup. The empty Behavior Tuning section was removed. No functional changes from 1.0.1-beta. |
| 1.0.1-beta | `suppress_during` replaced with an optional HA-startup peer-sync block, gated on the built-in Uptime integration. Three new inputs: `use_uptime_sensor` (boolean), `uptime_sensor` (entity, default `sensor.uptime`), `ha_start_block_seconds` (number, default 120). Box unchecked or sensor unavailable: no block, blueprint behaves the same as before for peer sync. Box checked and sensor reading: peer sync suppressed while uptime is below the threshold, with unit conversion for seconds, minutes, hours, or days based on the sensor's `unit_of_measurement` attribute. Reconcile branches are not blocked, because they are the intended response to boot drift. README: new section "A Small Note on the HA Start Block Window," worked example updated, parameters table updated. **Breaking**: users who had `suppress_during` pointed at an uptime sensor should enable `use_uptime_sensor` with a comparable `ha_start_block_seconds` value (120 to 180 is typical). Users who used `suppress_during` for maintenance windows or manual override modes lose that capability in this version. |
| 1.0.0-beta | Initial beta release. On/off sync only. Authority-on-reconcile model. Cross-domain entity selector (switches, lights, input_booleans, fans, groups). State equality breaks sync loops, so changes from physical presses, dashboards, automations, scenes, and voice all propagate. Mode restart so rapid toggles resolve to the latest state. HA start and optional reconcile-trigger-sensor branches for restart and bridge-reconnect drift recovery. Suppress-during gate. Debug logging. |
