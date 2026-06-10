# Linked Entities Pro (Beta)

A Home Assistant automation blueprint that keeps two or more entities in lockstep, with a deterministic tiebreaker for the moments when the group can briefly disagree.

When a wall switch and a smart bulb are wired to control the same light, or a Zigbee relay and a Tasmota module sit on the same lamp, the moment they disagree the next press lands wrong. The dashboard shows one thing, the fixture does another, and the room is broken until someone notices. The naive bidirectional sync automation either ping-pongs in a loop, or carries a recursion guard so loose that it fires intermittently. *Linked Entities Pro* fixes both. A strict three-condition context filter cleanly separates a real human press from an automated update, so the followers' state changes never trip the trigger again. Any number of entities can sit in one linked group, in any mix of integrations, and a change on any of them propagates to the others.

Two events can leave the group out of sync. On Home Assistant restart, entities come back online in unpredictable order, some still carrying stale states from before. On a Zigbee2MQTT bridge reconnect, the bridge republishes every device's last-known state at once, which looks like every switch in the house just got pressed by an invisible hand. In both moments the linked entities can disagree, and the blueprint cannot tell from the events alone which state represents reality. That is where the *authority entity* comes in. You designate one entity in the group as the source of truth for these moments. When a reconcile event fires, the authority's current state is treated as correct and the others are commanded into alignment. During ordinary use the authority has no special role: any linked entity can drive the others. It only matters in those two narrow situations.

Full write-up and the longer story behind the design: <https://xeazy.com/linked-entity-pro/>  Questions and discussion: <https://xeazy.com/logbook/d/38-linked-entities-pro-blueprint>

## What You Get

- Bidirectional sync across two or more entities. A change on any of them propagates to the others.
- Cross-domain support. Switches, lights, input_booleans, fans, and groups can mix freely in the same linked group.
- A three-condition context filter that distinguishes physical presses from automated changes, so the followers' updates do not loop back as new triggers.
- A reconcile branch that runs on Home Assistant start and on an optional bridge sensor reporting back online, with a designated authority entity acting as the tiebreaker.
- An optional suppress-during binary sensor that pauses the automation during maintenance windows or reboot indicators.
- Debug logging that can be toggled on during setup and off in normal operation.

Trigger filtering on the linked entities is restricted to `to: ['on', 'off']`, so devices coming back online from `unavailable` are not treated as physical presses. The reconcile branch waits up to 30 seconds for the authority to enter a known state before acting, so an unavailable authority on boot does not corrupt the alignment.

## Where This Fits

There are a few common ways to keep entities in sync in Home Assistant, and each has its place.

A one-direction automation is the right tool when the dependency really is one-way. A motion sensor triggers a light; the light has no reason to ever drive the sensor. Cheap and obvious.

A pair of mirrored automations, one for A drives B and one for B drives A, covers the bidirectional case. Easy to write, easy to break. Without a recursion guard you get a ping-pong loop. With a naive guard you get a loop that fires intermittently. With a strict guard you get drift on restart that nothing resolves. Most published switch-sync blueprints sit here.

Native Home Assistant Light Groups handle the n-entity case when every member is in the `light` domain. The group acts as the controllable parent. What groups do not give you: cross-domain mixing (a wall switch and a relay are not lights), or alignment of the underlying entities after a bridge reconnect that republishes them out of order.

This blueprint sits where you want bidirectional sync across two or more entities, want it to survive restarts and bridge reconnects without drift, and want to mix domains (switches, lights, input_booleans, fans, groups) freely.

## Requirements

Home Assistant 2026.6.0 or newer. That is the version it is verified on.

## Step 1: Import the Blueprint

The quick way, one click:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTheThinkingHome%2FAutomations%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Flinked_entities_pro_beta.yaml)

Or by hand:

1. Go to Settings, then Automations & Scenes, then the Blueprints tab.
2. Click Import Blueprint at the bottom right.
3. Paste this URL and click Preview:

```
https://github.com/TheThinkingHome/Automations/blob/main/blueprints/automation/linked_entities_pro_beta.yaml
```

4. Confirm and Import.

After import, the blueprint appears on the Blueprints page with a Create Automation button next to it.

## Step 2: Create the Automation

Click Create Automation on the imported blueprint. Home Assistant opens the automation editor with the blueprint's inputs grouped into four sections.

- **📦 Linked Entities**: pick the entities you want to keep in sync, and optionally designate one as the authority.
- **🔄 Reconciliation**: turn the Home Assistant start reconcile on or off, and optionally point at a bridge sensor (Zigbee2MQTT users want `binary_sensor.zigbee2mqtt_running`).
- **⚙️ Behavior Tuning**: set the sync delay between dispatched actions and optionally point at a suppress-during sensor.
- **🛠 Advanced**: toggle debug logging while you confirm the setup.

Give the automation a name, save it, and the sync is live.

The full parameter reference is in the table at the end of this page.

## A Small Note on Physical Presses

The peer sync branch has to tell a real physical press apart from an automated state change, or it would loop. The blueprint uses a strict three-condition filter on the trigger context: the event has a context id, has no parent context id, and has no user id. A physical press from a switch (or any source that arrives outside an automation) satisfies all three. An automation-driven change carries this blueprint's own context as its parent id, which fails the second condition. A change from the Home Assistant frontend carries a user id, which fails the third.

The result is clean: presses propagate, the followers' updates do not propagate further. Most published sync blueprints check only the parent id, which lets some edge cases through; this one closes the door on all three at once.

## A Small Note on Restart and Bridge Drift

Two events sit outside the peer sync model and have their own branch.

A Home Assistant restart brings entities back up in unpredictable order. Some report stale states from before the restart, some come up unavailable for a while and then settle, and the linked group can disagree for several seconds while everything finishes loading. The reconcile branch reads the authority entity's state, waits up to 30 seconds for it to enter a known on or off state, then commands every other entity in the group to match. If the authority is still unavailable after the 30-second wait, the reconcile exits cleanly and waits for the next event to try again.

A Zigbee2MQTT bridge reconnect is the second event. Z2M republishes every device's last-known state on reconnect with a null parent context, which is indistinguishable from a physical press at the event level. Without a guard, that republish cascade can drive the followers in random directions. The reconcile trigger sensor input solves this: point it at `binary_sensor.zigbee2mqtt_running`, and the reconcile branch runs on its off-to-on transition, treating the authority as truth and aligning everything else.

Users of other Zigbee integrations or other transports can usually leave the reconcile trigger sensor blank: ZHA does not republish on coordinator reconnect, Z-Wave JS tracks node state internally and does not flood events, and WiFi devices reconnect per device, not as a cascade. See the table further down.

## A Small Note on Rapid Presses

The automation runs in `mode: single` with `max_exceeded: silent`. If a second physical press arrives while the first run is still dispatching, the second is dropped silently rather than racing the first. The followers end up at the state of the first press. A short pause between presses (a fraction of a second is enough on most hardware) is the user-visible workaround.

This is a deliberate trade. The alternative, `mode: queued`, would honor every press in order but lets the followers fall behind on a flurry of toggles, which is its own kind of confusing. Single mode keeps the followers truthful to the most recent settled state and drops the redundant ones, which is the right default for a sync automation.

## Worked Example

A common one: a fixture is controlled by two physical things, a Zigbee wall switch and a WiFi relay (a Tasmota module wired behind the cabinet, closer to the actual load). Either can turn the light on or off, and both have to stay in sync, because the next press of the wall switch will go the wrong way if the two report different states.

Configure the blueprint like this:

- **Linked Entities**: `switch.kitchen_counter_wall_switch`, `switch.kitchen_counter_relay`
- **Authority Entity**: `switch.kitchen_counter_relay` (closer to the load, so it carries the true state if the two disagree)
- **Reconcile on Home Assistant Start**: ON
- **Reconcile Trigger Sensor**: `binary_sensor.zigbee2mqtt_running`
- **Sync Delay**: 0 (a two-entity pair has no mesh-load risk, so no throttling is needed)
- **Suppress During**: blank
- **Debug Logging**: ON for the first day, OFF after

Press the wall switch, the relay follows. Toggle the relay from the dashboard, the wall switch follows. Restart Home Assistant, the wall switch is pulled into alignment with the relay. Cycle the Z2M addon, same thing. The two states never drift apart.

For more worked examples, including mixed-integration groups and the longer story behind each design choice, see the article: <https://xeazy.com/linked-entity-pro/>

## Reconcile Trigger Sensor by Integration

| Integration | Recommended sensor | Why |
| --- | --- | --- |
| Zigbee2MQTT | `binary_sensor.zigbee2mqtt_running` | Z2M republishes every device's state on bridge reconnect with `parent_id=null`, indistinguishable from physical presses. This is the canonical use case. |
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
| `sync_delay_ms` | No | `200` | a whole number of milliseconds, 0 to 5000 | Milliseconds between dispatched actions. Set to 0 for small pairs on fast hardware. Increase on busy meshes. |
| `suppress_during` | No | none | any `binary_sensor` | When this sensor is on, the automation is suppressed entirely. Useful for maintenance windows or reboot indicators. |
| `debug_enabled` | No | `false` | `true`, `false` | When true, writes a log line for every trigger and action. Turn off in normal operation. |

## Beta Status

This is a public preview. The blueprint runs in production at the original home it was built for, with two pairs verified: a Zigbee wall switch and a Tasmota relay (cross-integration), and two Z2M switches on the same fixture (same-integration). What it has not yet been is validated by an external tester on a different setup. Once an external tester confirms it works in their environment, the blueprint graduates from beta. At that point the file is republished at `blueprints/automation/linked_entities_pro.yaml` (without the `_beta` suffix) and the version becomes 1.0.0 stable.

Want to be that tester? [Drop into the forum thread](https://xeazy.com/logbook/d/38-linked-entities-pro-blueprint) and say hi.

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>)

This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.
