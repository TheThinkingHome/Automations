# Blueprints Exchange opening post: Linked Entities Pro

**Draft v1.30** | Last updated: mode: single objection reframed — wait_template jargon dropped, "is worse" replaced with conscious design choice framing ("detect every toggle"), final beat shortened

Working markdown for iteration before publishing to community.home-assistant.io. Version bumps each time Claude saves so you can tell fresh from stale at a glance.

---

## Suggested thread title

**Linked Entities Pro: Keep entities synced. No loops. No drift. No recursion-guard blind spots.**

---

## Post body

The kitchen counter fixture in my house has two controllers: a Zigbee wall switch by the doorway and a Tasmota relay behind the cabinet. They kept drifting out of sync. The wall switch would say on, the lights would be off. The next press always lands wrong.

I built this sync three times before getting it right. The first version was a pair of mirrored automations that worked until it looped. The second added a context filter that asked *"did a human do that?"* It worked until a dashboard toggle or a scene quietly slipped past it. And no matter which version was running, every Home Assistant restart left the two controllers settling on whatever stale state they had before.

**Linked Entities Pro** is the third version, generalized to any number of entities and any mix of domains. It is packaged as a blueprint so you can skip the first two.

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTheThinkingHome%2FAutomations%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Flinked_entities_pro.yaml)

Full README and parameter reference: <https://github.com/TheThinkingHome/Automations/blob/main/blueprints/automation/linked_entities_pro.md>

## What's different about this approach

The blueprint replaces the recursion guard with a **state-equality gate**. The automation is recursive on purpose. Every state change on a linked entity triggers the automation, which runs in `mode: restart`. Each new trigger cancels any run still in flight. The gate asks one question of the other linked entities: do they all match now? If yes, the run stops. No command, no loop. If no, it dispatches another command. The followers respond, their new state changes trigger the automation again, and the cycle continues until everyone agrees.

The gate is *context-blind*: it does not care who caused the change. A physical press, a dashboard toggle, a scene automation, and a voice command all flow through the same path and propagate the same way. And because the gate's only question is whether the group agrees yet, the same mechanism scales: two entities or a dozen, switches and lights and input_booleans and fans and groups mixed in the same linked group, all kept in lockstep.

Restart drift is handled separately. A designated **authority entity** wins the tiebreaker. After a restart, the group settles on truth instead of stale state.

## What You Can Configure

* **Linked entities.** Two or more, any mix of switches, lights, input_booleans, fans, and groups in the same linked group.
* **Authority entity.** One entity wins the tiebreaker after Home Assistant restarts and bridge reconnects. During ordinary use, any entity can drive the others.
* **Restart reconcile.** A reconcile branch runs on Home Assistant start, and optionally on a bridge-online sensor coming back online (Zigbee2MQTT users: `binary_sensor.zigbee2mqtt_running`; ZHA, Z-Wave JS, and WiFi users can leave blank), to pull the group into alignment with the authority.
* **HA-startup peer-sync block.** Gate on the built-in Uptime integration to suppress peer sync for a configurable window after HA starts.
* **Sync delay.** Pause between trigger and dispatched commands. Doubles as the cancellation window for rapid opposite presses and as the cascade absorber on larger groups.
* **Debug logging.** Log lines for every trigger and action. Useful while confirming the setup.

**What's in scope?** On/off state only, by design. Brightness, color, and fan speed do not propagate, but lights and fans can still be linked for their on/off behavior. If there's enough interest in a domain-specific companion (a Linked Lights with brightness and color), I'll build it as a separate blueprint, since each domain handles attributes differently.

## Worked example: a four-entity passageway

Three hallway lights on separate switches (upstairs, downstairs, stairwell) plus an `input_boolean` helper that the dashboard reads as "is the passageway lit." All four kept in sync:

```yaml
automation:
  - alias: "Passageway Lights Sync"
    use_blueprint:
      path: TheThinkingHome/linked_entities_pro.yaml
      input:
        linked_entities:
          - input_boolean.passageway_lights
          - light.upstairs_hall
          - light.downstairs_hall
          - light.stairwell
        authority_entity: input_boolean.passageway_lights
        reconcile_on_ha_start: true
        reconcile_trigger_sensor: binary_sensor.zigbee2mqtt_running
        use_uptime_sensor: true
        uptime_sensor: sensor.uptime
        ha_start_block_seconds: 120
        sync_delay_ms: 200
```

The `input_boolean` is the authority because helpers restore cleanly across restarts. The dashboard tile reads the helper. Tap any switch or toggle the helper from the dashboard, and the others follow. After a restart, the lights are pulled into alignment with whatever state the helper restored to.

## Common questions

* **How do you tune sync delay?** For fast hardware and small groups, a delay close to 0ms might work. For slower hardware or larger groups, the default 200ms absorbs the response cascade. To find the right value in between, turn on debug logging and watch your logs.

* **Why not just use a group?** Two problems. First, native groups are typed: a Light Group takes only lights, a switch group only switches, and a wall switch, a Tasmota relay, and an input_boolean cannot share one. Second, even when the members match, the group only flows one way. Toggle the group and the members follow. Toggle a member directly and the siblings stay where they were. A group reflects state. It does not keep independent controllers in sync.

* **Why not use `mode: single`?** Two problems. First, naive `single` floods larger groups: each follower reports back at a different moment, the run re-checks on the first arrival, sees a mismatch, and dispatches again. Second, the obvious fix is pushing back automation termination until the group agrees, but then any press during the wait gets ignored. By design, we want to detect every toggle. `mode: restart` does both: a new trigger cancels the run in flight, the latest press wins, and the cascade collapses to agreement.

* **What about unavailable or frozen devices?** Unknown and unavailable are filtered from the start: the trigger fires only on `on`/`off` transitions, and the dispatch excludes any entity in `unavailable` or `unknown`. Frozen devices cannot be caught, but they cause no harm. The available devices settle, and the automation terminates without error.

Version 1.0.0 stable as of June 2026.

Questions, bug reports, and use cases I didn't think of are welcome here. Half the fun of this is seeing where other people aim it.
