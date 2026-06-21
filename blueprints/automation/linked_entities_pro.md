# Linked Entities Pro

*Linked Entities Pro* is a Home Assistant blueprint for two-way sync, generalized to any number of entities and any mix of domains: switches, lights, input_booleans, fans, and groups. One blueprint, every linked group in the house.

Picture two independent switches, a switch for the hallway lights and a switch for the stairway lights. Anyone walking through presses both. Now picture a wall switch wired to the lights, paired with a second virtual switch across the room. Or picture a smart bulb that stays powered all the time, controlled by another virtual wall switch that sends events instead of controlling the bulb directly.

In each of these setups, the controllers can fall out of sync. When they do, the next press lands wrong or gets ignored, the dashboard shows one thing while the fixture does another, and the room is broken until someone notices. The naive two-way sync automation either ping-pongs in a loop, or carries a recursion guard so loose that it fires intermittently.

*Linked Entities Pro* fixes both. A state-equality gate stops every run when the other linked entities match the source's new state, so the followers' responses cannot drive the loop any further.

Full write-up and the longer story behind the design: <https://xeazy.com/linked-entity-pro/>  Questions and discussion: <https://xeazy.com/logbook/d/38-linked-entities-pro-blueprint>

## What You Get

- Two-way sync across two or more entities. A change on any of them propagates to the others.
- Cross-domain support. Switches, lights, input_booleans, fans, and groups can mix freely in the same linked group.
- A state-equality gate that stops a run when the other linked entities match the source's new state, so the followers' responses cannot loop the trigger back on itself.
- A reconcile branch that runs on Home Assistant start and on an optional bridge sensor reporting back online, with a designated authority entity acting as the tiebreaker.
- An optional peer-sync block for the first minute or two after Home Assistant starts, gated on the built-in Uptime integration. Prevents republished stale states from propagating during boot. The reconcile branches still run.
- Debug logging that can be toggled on during setup and off in normal operation.

## Where This Fits

There are a few common ways to keep entities in sync in Home Assistant, and each has its place.

A one-direction automation is the right tool when the dependency really is one-way. A motion sensor triggers a light, and the light has no reason to ever drive the sensor. This approach is cheap and obvious.

A pair of mirrored automations, one for A drives B and one for B drives A, covers the two-way case. They are easy to write and easy to break. Without a recursion guard you get a ping-pong loop. With a naive guard you get a loop that fires intermittently. With a strict guard you get drift on restart that nothing resolves. Most published switch-sync blueprints sit here.

Native Home Assistant Light Groups handle groups of any size when every member is in the `light` domain. The group acts as the controllable parent: turn the group on and every member turns on. But the sync only flows downward, from the group to its members. Change one member directly, from its own wall switch or its own card, and the siblings do not follow. Groups also do not give you cross-domain mixing (a wall switch and a relay are not lights), or alignment of the underlying entities after a bridge reconnect that republishes them out of order.

This blueprint sits where pairs of mirrored automations break and Light Groups don't reach: bidirectional, cross-domain, and resilient to restarts and bridge reconnects.

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

## How It Works

The design is recursive on purpose. When any linked entity changes state, the automation checks whether the others already match. If they do, the run ends. If they do not, it waits `sync_delay_ms`, then commands the rest of the group to match. Each follower that responds fires the trigger again, and another run starts. `mode: restart` makes this rhythm work cleanly: if a new trigger arrives during the wait (a third or fourth follower reporting in, or a fresh press of any entity), it cancels the in-flight run and starts fresh on the latest state. By the second or third pass, enough followers have aligned that the state-equality gate finds them matching and exits silently. There is no explicit recursion guard; the gate is the convergence path.

Two events fall outside this peer model and run a separate reconcile branch. The first is a Home Assistant restart, when entities come back up in unpredictable order with potentially stale states. The second is a Zigbee2MQTT bridge reconnect, when Z2M republishes every device's last-known state in a cascade. ZHA, Z-Wave, and WiFi integrations do not have this republish-on-reconnect behavior, so if you are not using Z2M, leave the reconcile trigger sensor input blank. When the branch fires, it reads the authority entity's state and commands the rest of the group to match. An optional HA start block (gated on the Uptime integration) suppresses peer sync during boot so stale republished states do not cascade through the peer model before the reconcile has caught up.

### Why a sync delay, and why it runs first

Past a pair, the followers' responses don't land together, so without a pause each one re-triggers the run mid-settle and each run sends another round of commands.

The delay lets the group settle before the next command leaves, folding the re-triggers into one clean convergence instead of a flood. On a ten-entity group that's a handful of commands per press instead of dozens, and on a Zigbee mesh that flood congests the network and can drop a command, leaving a device behind.

It runs first because `mode: restart` can only cancel a command that hasn't been sent yet. With the delay in front, the follower re-triggers (and any new press) cancel the run before it commands anything, so only the settled state is ever sent. Put it after the dispatch and every run fires immediately, each re-trigger fires another, and the flood comes back.

## Limitations

- **On/off only.** Brightness, color, and fan speed do not propagate. Light and fan domains are supported because their on/off state still syncs cleanly. A domain-specific Linked Lights companion may follow if there is enough interest.
- **Frozen devices are not detected.** A device reporting a stale state and not responding to commands (without dropping to `unavailable`) will stay out of alignment until something else gets it responding again. From Home Assistant's perspective the device looks fine, so the blueprint cannot detect the failure.
- **Unavailable and unknown states are filtered.** Entities in those states neither fire the trigger nor receive dispatched commands. When they return to a known state, they rejoin the group naturally.

## Worked Example

A common one: two bedside lamps, one on each side of the bed, each with a smart bulb. Either can be turned on or off independently from a tap on the dashboard, a voice command, or a Zigbee remote on the nightstand, and both should stay in sync so the pair always shows the same state.

Configure the blueprint like this:

- **Linked Entities**: `light.bedside_lamp_left`, `light.bedside_lamp_right`
- **Authority Entity**: `light.bedside_lamp_left` (either is fine; they are peers)
- **Reconcile on Home Assistant Start**: ON
- **Reconcile Trigger Sensor**: `binary_sensor.zigbee2mqtt_running` if the bulbs are on Z2M; leave blank otherwise
- **Block Peer Sync During HA Startup**: ON, if you have the Uptime integration installed
- **Uptime Sensor**: `sensor.uptime`
- **Block Peer Sync After HA Start (seconds)**: 120
- **Sync Delay**: 0 (a two-entity pair has no mesh-load risk, so throttling is not needed)
- **Debug Logging**: ON for the first day, OFF after

Tap left and right follows. Tap right and left follows. Ask Alexa to turn off the bedroom lamps and both go off together. Restart Home Assistant and the pair is pulled into alignment. The two states never drift apart.

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

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>)

This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.

## Changelog

| Version | Notes |
| --- | --- |
| 1.0.0 | Initial stable release. State-equality gate two-way sync across any number of entities and any mix of supported domains (switch, light, input_boolean, fan, group). Reconcile branch on HA start and on an optional reconcile trigger sensor, with a designated authority entity as the tiebreaker. Optional HA-startup peer-sync block via the Uptime integration. Tunable `sync_delay_ms` (default 200 ms). Verified on two production pairs and thirty simulator scenarios at group sizes 2 through 10. Beta history (1.0.0-beta through 1.0.3-beta) archived in the project document. |
