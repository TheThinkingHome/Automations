# System Stability

A Home Assistant template blueprint that creates a binary sensor reading `on` once Home Assistant has fully settled after a restart, and `unknown` while the system is still coming up. Built on the `homeassistant: start` event with `delay_on` handling the timed transition to stable.

After every HA restart there is a short window where things are settling. Integrations come online one by one, entities flicker between `unavailable` and their real states, MQTT-backed devices republish their states, presence sensors take a moment to find their occupants. Automations that fire on those state changes are firing on noise, not on real events. A motion-light automation that watches a presence sensor sees the sensor flip on (it just came up) and turns on the lights at 4 AM. A scene that reacts to `input_select.scene` changes sees the value bounce as helpers restore and triggers itself for no reason. A security automation watching for doors going from closed to open sees a contact sensor flicker as Zigbee re-establishes and treats it as an intrusion.

This blueprint creates a single binary sensor whose only job is to answer one question for the rest of your config: is the system fully stable? While the answer is anything other than `on` (typically `unknown` during boot and the settling window), dependent automations short-circuit themselves; only when it reads `on` should they run normally.

Full write-up and the longer story behind the design: <https://xeazy.com/system-stability-template-blueprint/>  Questions and discussion: <https://xeazy.com/logbook/>

## What You Get

- A binary sensor that reads `unknown` while HA is loading and during the settling window, then `on` once the system is stable. It stays `on` for the rest of the session, until the next HA restart.
- A single trigger: the `homeassistant: start` event. HA fires it once per boot, after every integration has finished setup.
- `delay_on` based timing. After the trigger fires, the sensor waits the configured number of seconds before transitioning to `on`. The settling period covers the residual delay between HA reporting itself as up and entities actually being stable.
- An adjustable window. 60 seconds is the default, which covers residual settling on most systems. Raise it for slower systems, lower it for faster ones.
- A unique_id so the sensor is registered and can be renamed or placed in an area from the UI.

The sensor is `on` only when the system is fully stable. Use it as a condition (run only when on), as a trigger (run when the system has just finished settling), or as a wait point (`wait_for_trigger to: 'on'`).

## Where This Fits

The closest built-in approach is to gate each automation on the `homeassistant: start` event directly: trigger on start, wait N seconds, then run the rest. It works for a single automation, but the boot window is a property of the system, not of any one automation. Once you have ten automations that need the gate, you have ten copies of the same delay buried in ten places, and changing the window means editing ten files.

A hand-written template sensor that does `now() - sensor.uptime` arithmetic also works, but it polls every minute and the math gets re-evaluated forever, even at 3 AM when the system has been up for sixteen hours. That is fine, but it is wasteful for something that has a definite end the moment the delay completes.

This blueprint sits where you want a single system-level boolean every other automation can read. It is event-driven (no polling) and sleeps the moment the delay completes.

## Requirements

Home Assistant 2024.6.0 or newer. No other integrations required; the blueprint uses only HA's built-in `homeassistant: start` event.

## Step 1: Import the Blueprint

The quick way, one click:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTheThinkingHome%2FAutomations%2Fblob%2Fmain%2Fblueprints%2Ftemplate%2Fsystem_stability.yaml)

Or by hand:

1. Go to Settings, then Automations & Scenes, then the Blueprints tab.
2. Click Import Blueprint at the bottom right.
3. Paste this URL and click Preview:

```
https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/system_stability.yaml
```

4. Confirm and Import.

After import, the blueprint appears on the Blueprints page.

## Step 2: The Catch with Template Blueprints

Automation blueprints get a friendly Create button right on the Blueprints page. Template blueprints do not. You will not find System Stability anywhere in the Create Helper list, and that is expected, not a fault. Template blueprints become real entities through a short piece of YAML called `use_blueprint`. It takes a path to the blueprint and a set of inputs:

```yaml
use_blueprint:
  path: TheThinkingHome/system_stability.yaml
  input:
    stability_delay_seconds: 60
```

`path` is relative to `config/blueprints/template/`, so it is the `<author>` folder from Step 1 plus the filename. `stability_delay_seconds` has a sensible default of 60 and can be omitted, in which case the entire `input:` block can be omitted too.

That block has to sit under Home Assistant's `template:` configuration. The tidy way to do that, and the way that scales once you make more sensors built from blueprints, is a package.

## Step 3: What a Package Is, and How to Create One

A package is a single YAML file that holds a bundle of related configuration. Rather than scattering a sensor here and an automation there across your main files, you put everything for one feature into one file, and Home Assistant folds it into your overall configuration at startup. Packages are optional, but they keep related things together and easy to find.

If you have never used packages, switch them on once. In your `configuration.yaml`, add the two lines below. If you already have a `homeassistant:` section, just add the `packages:` line underneath it:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Then create a folder named `packages` next to your `configuration.yaml`. Every YAML file you drop in there is now a package.

Now make a package file, for example `packages/system_stability.yaml`, and put your `use_blueprint` block inside a `template:` list:

```yaml
template:
  - use_blueprint:
      path: TheThinkingHome/system_stability.yaml
      input:
        stability_delay_seconds: 60
```

If you already keep template entities in your `configuration.yaml` under a `template:` key, you can drop the `use_blueprint` block into that list instead. The package is simply the cleaner home once you have more than one or two.

## Step 4: Load It

A brand-new package file needs a full Home Assistant restart to register the first time. After that, changes to the file only need a quick reload through Developer Tools, YAML, Reload Template Entities. If you put the block in `configuration.yaml` directly under an existing `template:` key, you can skip the restart and just reload template entities.

When it comes up you will have a `binary_sensor.system_stable`. Watch it in Developer Tools, States: it will be `unknown` until the first time HA fully restarts. On the next HA restart, you will see it stay `unknown` during the loading phase, stay `unknown` during the `delay_on` window after `homeassistant: start` fires (typically when the "Home Assistant has restarted" toast appears at the bottom of the UI), and then flip to `on`. From that point it reads `on` for the rest of the session, until the next HA restart. That is the full lifecycle.

## A Small Note on Why the System Is Not Stable Yet

Even after Home Assistant fires the `homeassistant: start` event, the system is not entirely stable. The heaviest state churn (integration setup, automation registration, entity creation) is over by then, but residual settling continues for another 30 to 60 seconds: Zigbee2MQTT republishing device states across the mesh, MQTT brokers reconnecting and replaying retained messages, presence sensors finding their occupants, contact sensors flickering as Zigbee re-establishes.

A critical automation that fires on those state changes is firing on noise, not on real events. A light automation watching a presence sensor sees the sensor flip on (it just came up) and turns on the lights at 4 AM. A scene dispatcher sees `input_select.scene` bounce as helpers restore and triggers itself for no reason. A security automation watching for doors going from closed to open sees a contact sensor flicker as Zigbee re-establishes and treats it as an intrusion. A power-cycle recovery automation sees a sensor briefly unavailable and cycles the plug it is supposed to protect.

The blueprint's job is to give every dependent a single system-wide signal that says "the system is fully stable, run normally." Light automations watching presence sensors wait. Scene dispatchers wait. Security monitoring waits. Power-cycle recovery automations skip the round they would otherwise run on a sensor that is briefly unavailable simply because Zigbee has not finished republishing.

## A Small Note on the Initial Unknown State

Trigger-based template entities in Home Assistant start in the `unknown` state and stay there until the trigger fires AND the `delay_on` window elapses. There is a window between HA boot and `homeassistant: start` firing where the binary sensor is `unknown`, plus the `delay_on` window after that during which the sensor is still `unknown` (waiting for residual settling). On a default setup the sensor reads `unknown` for roughly two minutes total wall-clock from boot, then transitions to `on`.

The correct pattern for dependents is to use `state == 'on'` as the precondition to proceed. `unknown` does not match `'on'`, so dependents stay blocked through both the loading phase and the settling window. The sensor never transitions back to `unknown` during normal operation, so once it reads `on`, dependents can rely on that signal until the next HA restart.

Two equivalent ways to write the precondition:

```yaml
condition: state
entity_id: binary_sensor.system_stable
state: 'on'
```

```yaml
condition: template
value_template: "{{ states('binary_sensor.system_stable') == 'on' }}"
```

Both read as "the system has fully settled, run normally." They block when the state is `'unknown'` (the loading phase and the settling window), and they would also block in the edge case where the state was `'unavailable'`. That is the right behavior: only proceed when the gate has explicitly cleared.

You will also see the sensor read `unknown` after a template integration reload (Developer Tools, YAML, Reload Template Entities) without an HA restart. The reload rebuilds the entity, the trigger has not fired since the reload, so the state goes back to `unknown` until the next HA restart fires `homeassistant: start` and the `delay_on` countdown completes. The same precondition pattern handles this correctly: dependents stay blocked, which is conservative but safe.

## A Small Note on the Window Size

60 seconds is the default and a good starting point for most systems. It covers the residual settling that happens after `homeassistant: start` fires: Z2M republishing device states, MQTT reconnects, presence sensors finding their occupants. The heaviest state churn is over before `homeassistant: start` fires; the window only needs to cover what comes after.

If your residual settling is faster than that and you want the gate to clear sooner, drop the window to 30 or 45 seconds. If you have a large Zigbee mesh, a slow MQTT broker, or anything else that takes longer to settle, raise it. The blueprint accepts values between 15 and 300 seconds.

A practical way to tune it: after a restart, watch the HA log for the moment the "Home Assistant has restarted" toast appears at the bottom of the UI. That is `homeassistant: start` firing. From that point, watch how long it takes for entities to stop bouncing between states. Whatever that interval is on your system, that is the right window, with a small margin for headroom.

## Worked Example

The package YAML for the default case:

```yaml
template:
  - use_blueprint:
      path: TheThinkingHome/system_stability.yaml
      input:
        stability_delay_seconds: 60
```

This creates `binary_sensor.system_stable`, which stays `unknown` during the loading phase, stays `unknown` for the 60 seconds after `homeassistant: start` fires (typically up to a minute after boot, once every integration has finished loading), then transitions to `on`. Combined with the loading phase, dependents are blocked for roughly two minutes total wall-clock on a default setup.

A typical use in a downstream automation, as a condition that only allows the automation to run once the system is fully stable:

```yaml
conditions:
  - alias: System is fully stable
    condition: state
    entity_id: binary_sensor.system_stable
    state: 'on'
```

A waiter that holds an automation until the system has finished settling, with a timeout so the automation does not hang forever if something goes wrong:

```yaml
- alias: Wait for the system to be stable
  wait_for_trigger:
    - trigger: state
      entity_id: binary_sensor.system_stable
      to: 'on'
  timeout: "00:03:00"
  continue_on_timeout: true
```

A trigger that runs an automation precisely when the system becomes stable (useful for sending a "system back up" notification, or for re-running a scene that should reflect the post-boot state of the house):

```yaml
triggers:
  - trigger: state
    entity_id: binary_sensor.system_stable
    to: 'on'
```

## Parameters

| Input | Required | Default | Accepted values | What it does |
| --- | --- | --- | --- | --- |
| `stability_delay_seconds` | No | `60` | integer between 15 and 300 | How many seconds after `homeassistant: start` fires before the sensor transitions to `on` (stable). This is the residual settling window: Z2M republish, MQTT reconnects, presence sensor wakeup. Default 60 seconds covers most systems. Raise for slower systems, lower for faster ones. |

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>)

This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.

## Changelog

| Version | Notes |
| --- | --- |
| 1.0.0 | Initial release. Trigger-based template binary_sensor fires once per HA boot from the `homeassistant: start` event. State template evaluates to `true`; `delay_on` holds the on transition for the configured settling window before the sensor reports `on` (stable). |
