# Linked Entities Pro (Beta)

> Keep two or more entities perfectly in sync, with a tiebreaker that wins on restart.

A free Home Assistant blueprint from [The Thinking Home](https://xeazy.com).

📖 [Read the article](https://xeazy.com/linked-entity-pro/) - the design and how it differs from other sync blueprints.
💬 [Discuss in the community](https://xeazy.com/logbook/d/38-linked-entities-pro-blueprint) - the forum thread for this blueprint.

---

## What it does

Link any number of entities together so they stay perfectly in sync. Switches, lights, fans, input_booleans, and groups can mix freely. A change on any entity propagates to the others.

When Home Assistant restarts, or when an optional bridge sensor reconnects, a designated authority entity acts as the tiebreaker. The authority's state is treated as truth, and the others are pulled into alignment. During normal user interaction, any linked entity can drive the others. The authority only matters when there's ambiguity to resolve.

Originally built to keep a Zigbee wall switch and a Tasmota relay in lockstep, but generalized to any number of entities and any mix of integrations.

## What sets it apart

The published switch-sync blueprints carry known weaknesses. The most-imported one has a recursion guard that fails about 30% of the time according to user comments, and it's switch-domain only. A more feature-rich alternative fires on every brightness tick during a fade and has no restart or reconnect safety. Neither has a defensible answer for what happens when devices come back online with stale or republished states.

Linked Entities Pro fixes all of that:

- A three-condition context filter that properly discriminates physical presses from automated changes. Loops do not happen.
- Cross-domain selector. Switches, lights, input_booleans, fans, and groups can mix freely in the same linked group.
- Trigger filters on `to: ['on', 'off']` only. Devices coming back from unavailable do not get treated as physical presses.
- Reconcile-on-Home-Assistant-start branch with deterministic authority-wins outcome. Boot-time chaos is solved.
- Optional reconcile trigger sensor for catching cascades after a bridge reconnect. Integration-agnostic, so Z2M, ZHA, Z-Wave, and WiFi users can wire whatever fits.
- Optional suppress-during binary sensor for maintenance windows or reboot indicators.
- Debug logging for setup confirmation.

## Import

One-click import (opens in your Home Assistant):

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTheThinkingHome%2FAutomations%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Flinked_entities_pro_beta.yaml)

Or paste this URL into Home Assistant's blueprint import dialog manually:

```
https://github.com/TheThinkingHome/Automations/blob/main/blueprints/automation/linked_entities_pro_beta.yaml
```

In Home Assistant, go to **Settings > Automations & scenes > Blueprints**, click **Import Blueprint**, paste the URL, and confirm.

Once imported, create a new automation from the blueprint and fill in the inputs below.

## Inputs

### 📦 Linked Entities

**Linked Entities** *(required)*. The entities you want to keep in sync. Pick two or more. Switches, lights, input_booleans, fans, and groups are all supported and can mix freely.

**Authority Entity** *(optional)*. The tiebreaker entity. Used only when Home Assistant restarts or the reconcile trigger sensor fires. During normal use, any linked entity can drive the others. If left blank, the first entity in the Linked Entities list is used as the authority.

### 🔄 Reconciliation

**Reconcile on Home Assistant Start** *(default ON)*. When Home Assistant starts up, reconcile linked entities to the authority's state. Devices come back online in unpredictable order during boot, and some report stale states. Leave on unless you have a reason to turn it off.

**Reconcile Trigger Sensor** *(optional)*. When this sensor goes from off to on, reconcile linked entities to the authority's state. Catches state cascades after a bridge reconnect. See the table below for integration-specific recommendations.

### ⚙️ Behavior Tuning

**Sync Delay** *(default 200 ms, range 0 to 5000)*. Milliseconds to wait between dispatching actions to followers. Helps Zigbee meshes and busy hubs avoid command storms when many entities sync at once. Increase if commands get dropped on large groups. Decrease toward 0 on fast hardware with a snappy mesh.

**Suppress During** *(optional)*. When this sensor is on, sync is suppressed entirely. Useful for maintenance windows, manual override modes, or pointing at a Home Assistant restart indicator.

### 🛠 Advanced

**Debug Logging** *(default OFF)*. Write detailed log lines to the system log on every trigger and action. Helpful for confirming behavior during setup. Noisy in normal operation, so turn off after setup.

## Worked example

Imagine a kitchen counter fixture controlled by two things: a Zigbee wall switch, and a WiFi relay (a Tasmota module behind the cabinet). Either one can turn the fixture on or off. Both have to stay in sync, because if the wall switch reports off while the relay actually has the lights on, the next press of the wall switch will go the wrong way.

Configure Linked Entities Pro like this:

- **Linked Entities**: `switch.kitchen_counter_wall_switch`, `switch.kitchen_counter_relay`
- **Authority Entity**: `switch.kitchen_counter_relay` (closer to the actual load, so it's the source of truth)
- **Reconcile on Home Assistant Start**: ON
- **Reconcile Trigger Sensor**: `binary_sensor.zigbee2mqtt_running` (if you're a Z2M user; otherwise blank)
- **Sync Delay**: 200
- **Suppress During**: blank
- **Debug Logging**: ON for first day, OFF after

That's it. Press the wall switch, the relay follows. Toggle the relay from the dashboard, the wall switch follows. Reboot Home Assistant, the wall switch is pulled into alignment with the relay. Cycle the Z2M addon, same thing. The lights and the switch state never drift apart.

## Reconcile trigger sensor by integration

| Integration | Recommended sensor | Why |
|---|---|---|
| Zigbee2MQTT | `binary_sensor.zigbee2mqtt_running` | Z2M republishes every device's state on bridge reconnect with `parent_id=null`, indistinguishable from physical presses. This is the canonical use case. |
| ZHA | Usually leave blank | ZHA does not republish states on coordinator reconnect. The HA start reconcile is enough. |
| Z-Wave JS | Usually leave blank | The driver tracks node state internally and does not flood events on reconnect. |
| WiFi (Shelly, Tasmota, etc.) | Usually leave blank | Reconnects are per-device, not cascaded. |
| Mixed | Pick the most failure-prone bridge | One sensor, used for the integration that causes the most drift. |

## Edge cases handled

- **Physical press vs. automated change**: a three-condition context filter (`context.id` not none AND `context.parent_id` is none AND `context.user_id` is none) discriminates between the two. The peer sync branch fires only on real physical presses, so the followers' state changes (which carry this automation's context as `parent_id`) get filtered out cleanly. Loops do not happen.
- **Devices coming back from unavailable**: trigger filters on `to: ['on', 'off']` only, so `unavailable -> on` transitions during reboot do not get treated as user actions.
- **Authority not ready on boot**: the reconcile branch waits up to 30 seconds for the authority to enter a known state before acting. If the authority is still unavailable after the wait, reconcile is skipped with a debug log entry and the automation exits cleanly.
- **Rapid double presses**: `mode: single` with `max_exceeded: silent`. The second press is dropped silently rather than racing the first. Followers end up at the state of the first press. A short pause between presses is the user-visible workaround.

## Versions

**1.0.0 (Beta)** - Initial beta release. On/off sync only. Authority-on-reconcile model. Cross-domain entity selector. Three-condition context filter for physical-press detection. HA start and optional reconcile-trigger-sensor branches for restart and bridge-reconnect drift recovery. Suppress-during gate. Debug logging.

### Roadmap

**1.1.0 (planned)** - Brightness and color temperature sync. Optional, default off, backwards compatible.

## Beta status

This is a public preview. It runs in production at the original home it was built for, but has not yet been validated by an external tester on a different setup. Once an external tester confirms it works in their environment, the blueprint graduates from beta. At that point the file is republished at `blueprints/automation/linked_entities_pro.yaml` (without the `_beta` suffix) and the version becomes 1.0.0 stable.

Want to be that tester? [Drop into the forum thread](https://xeazy.com/logbook/d/38-linked-entities-pro-blueprint) and say hi.

## License

Licensed under [GPL-3.0](LICENSE). Free to use, modify, and share, with the requirement that derivative works carry the same license. A link back to xeazy.com is appreciated but not required.

---

*Built by James Lander at [The Thinking Home](https://xeazy.com).*
