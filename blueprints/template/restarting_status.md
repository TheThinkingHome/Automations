# Restarting Status

A Home Assistant template blueprint that creates a binary sensor reading `on` while Home Assistant is still settling after a restart, then `off` once the system is stable. Built on the `homeassistant: start` event with `auto_off` handling the timed return to off.

After every HA restart there is a short window where things are settling. Integrations come online one by one, entities flicker between `unavailable` and their real states, MQTT-backed devices republish their states, presence sensors take a moment to find their occupants. Automations that fire on those state changes are firing on noise, not on real events. A motion-light automation that watches a presence sensor sees the sensor flip on (it just came up) and turns on the lights at 4 AM. A scene that reacts to `input_select.scene` changes sees the value bounce as helpers restore and triggers itself for no reason. A security automation watching for doors going from closed to open sees a contact sensor flicker as Zigbee re-establishes and treats it as an intrusion.

This blueprint creates a single binary sensor whose only job is to answer one question for the rest of your config: is the system still settling? While the answer is `on` (or `unknown`, which dependents should treat the same way), dependent automations can short-circuit themselves; only when it reads `off` should they run normally.

Full write-up and the longer story behind the design: <https://xeazy.com/restarting-status-template-blueprint/>  Questions and discussion: <https://xeazy.com/logbook/>

## What You Get

- A binary sensor that reads `on` from the moment HA finishes loading and stays on for the configured window, then flips to `off`.
- A single trigger: the `homeassistant: start` event. HA fires it once per boot, after every integration has finished setup.
- Auto-off based timing. After the configured window the sensor flips to `off` without any further evaluation. Lightest-weight implementation possible: active during the restart window, then sleeps until the next HA boot.
- An adjustable window. 60 seconds is the default, which covers residual settling on most systems. Raise it for slower systems, lower it for faster ones.
- A unique_id so the sensor is registered and can be renamed or placed in an area from the UI.

The sensor is `on` during the restart window and `off` afterwards. Use it as a condition (block the automation while on), as a trigger (run when the system has just finished settling), or as a wait point (`wait_for_trigger to: 'off'`).

## Where This Fits

The closest built-in approach is to gate each automation on the `homeassistant: start` event directly: trigger on start, wait N seconds, then run the rest. It works for a single automation, but the boot window is a property of the system, not of any one automation. Once you have ten automations that need the gate, you have ten copies of the same delay buried in ten places, and changing the window means editing ten files.

A hand-written template sensor that does `now() - sensor.uptime` arithmetic also works, but it polls every minute and the math gets re-evaluated forever, even at 3 AM when the system has been up for sixteen hours. That is fine, but it is wasteful for something that has a definite end the moment the timer expires.

This blueprint sits where you want a single system-level boolean every other automation can read. It is event-driven (no polling) and sleeps the moment the window closes.

## Requirements

Home Assistant 2024.6.0 or newer. No other integrations required; the blueprint uses only HA's built-in `homeassistant: start` event.

## Step 1: Import the Blueprint

The quick way, one click:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FTheThinkingHome%2FAutomations%2Fblob%2Fmain%2Fblueprints%2Ftemplate%2Frestarting_status.yaml)

Or by hand:

1. Go to Settings, then Automations & Scenes, then the Blueprints tab.
2. Click Import Blueprint at the bottom right.
3. Paste this URL and click Preview:

```
https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/restarting_status.yaml
```

4. Confirm and Import.

After import, the blueprint appears on the Blueprints page.

## Step 2: The Catch with Template Blueprints

Automation blueprints get a friendly Create button right on the Blueprints page. Template blueprints do not. You will not find Restarting Status anywhere in the Create Helper list, and that is expected, not a fault. Template blueprints become real entities through a short piece of YAML called `use_blueprint`. It takes a path to the blueprint and a set of inputs:

```yaml
use_blueprint:
  path: TheThinkingHome/restarting_status.yaml
  input:
    restart_window_seconds: 60
```

`path` is relative to `config/blueprints/template/`, so it is the `<author>` folder from Step 1 plus the filename. `restart_window_seconds` has a sensible default of 60 and can be omitted, in which case the entire `input:` block can be omitted too.

That block has to sit under Home Assistant's `template:` configuration. The tidy way to do that, and the way that scales once you make more sensors built from blueprints, is a package.

## Step 3: What a Package Is, and How to Create One

A package is a single YAML file that holds a bundle of related configuration. Rather than scattering a sensor here and an automation there across your main files, you put everything for one feature into one file, and Home Assistant folds it into your overall configuration at startup. Packages are optional, but they keep related things together and easy to find.

If you have never used packages, switch them on once. In your `configuration.yaml`, add the two lines below. If you already have a `homeassistant:` section, just add the `packages:` line underneath it:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Then create a folder named `packages` next to your `configuration.yaml`. Every YAML file you drop in there is now a package.

Now make a package file, for example `packages/restarting_status.yaml`, and put your `use_blueprint` block inside a `template:` list:

```yaml
template:
  - use_blueprint:
      path: TheThinkingHome/restarting_status.yaml
      input:
        restart_window_seconds: 60
```

If you already keep template entities in your `configuration.yaml` under a `template:` key, you can drop the `use_blueprint` block into that list instead. The package is simply the cleaner home once you have more than one or two.

## Step 4: Load It

A brand-new package file needs a full Home Assistant restart to register the first time. After that, changes to the file only need a quick reload through Developer Tools, YAML, Reload Template Entities. If you put the block in `configuration.yaml` directly under an existing `template:` key, you can skip the restart and just reload template entities.

When it comes up you will have a `binary_sensor.restarting_status`. Watch it in Developer Tools, States: it will be `unknown` until the first time HA fully restarts. On the next HA restart, you will see it stay `unknown` during the loading phase, flip to `on` the moment `homeassistant: start` fires (typically when the "Home Assistant has restarted" toast appears at the bottom of the UI), stay on for the configured window, then flip to `off`. That is the full lifecycle.

## A Small Note on Why the System Is Not Stable Yet

Even after Home Assistant fires the `homeassistant: start` event, the system is not entirely stable. The heaviest state churn (integration setup, automation registration, entity creation) is over by then, but residual settling continues for another 30 to 60 seconds: Zigbee2MQTT republishing device states across the mesh, MQTT brokers reconnecting and replaying retained messages, presence sensors finding their occupants, contact sensors flickering as Zigbee re-establishes.

A critical automation that fires on those state changes is firing on noise, not on real events. A light automation watching a presence sensor sees the sensor flip on (it just came up) and turns on the lights at 4 AM. A scene dispatcher sees `input_select.scene` bounce as helpers restore and triggers itself for no reason. A security automation watching for doors going from closed to open sees a contact sensor flicker as Zigbee re-establishes and treats it as an intrusion. A power-cycle recovery automation sees a sensor briefly unavailable and cycles the plug it is supposed to protect.

The blueprint's job is to give every dependent a single system-wide signal that says "things are still settling, do not act yet." Light automations watching presence sensors hold their fire. Scene dispatchers wait. Security monitoring stays muted. Power-cycle recovery automations skip the round they would otherwise run on a sensor that is briefly unavailable simply because Zigbee has not finished republishing.

## A Small Note on the Initial Unknown State

Trigger-based template entities in Home Assistant start in the `unknown` state and stay there until the trigger fires for the first time. There is a window between HA boot and `homeassistant: start` firing where the binary sensor is `unknown` rather than `on`. On a typical system this window is up to a minute, sometimes longer; it is exactly the loading phase when integrations and automations are coming up one by one.

The correct pattern for dependents is to use `state == 'off'` as the release condition. Reading the gate this way, `unknown` does not match `'off'`, so dependents stay blocked through the loading phase as well as through the `on` window. The net effect is that dependents are blocked from boot through the loading phase plus the configured window, which is roughly two minutes total wall-clock on a default setup.

Two equivalent ways to write the release condition:

```yaml
condition: state
entity_id: binary_sensor.restarting_status
state: 'off'
```

```yaml
condition: template
value_template: "{{ states('binary_sensor.restarting_status') == 'off' }}"
```

Both read as "the system has fully settled, run normally." They block when the state is `'on'` (the active window), and they also block when the state is `'unknown'` or `'unavailable'`. That is the right behavior: until the gate has explicitly cleared, do not act.

You will also see the sensor read `unknown` after a template integration reload (Developer Tools, YAML, Reload Template Entities) without an HA restart. The reload rebuilds the entity and its state goes back to `unknown` until the next boot. The same release-condition pattern handles this correctly: dependents stay blocked, which is conservative but safe.

## A Small Note on the Window Size

60 seconds is the default and a good starting point for most systems. It covers the residual settling that happens after `homeassistant: start` fires: Z2M republishing device states, MQTT reconnects, presence sensors finding their occupants. The heaviest state churn is over before `homeassistant: start` fires; the window only needs to cover what comes after.

If your residual settling is faster than that and you want the gate to release sooner, drop the window to 30 or 45 seconds. If you have a large Zigbee mesh, a slow MQTT broker, or anything else that takes longer to settle, raise it. The blueprint accepts values between 15 and 300 seconds.

A practical way to tune it: after a restart, watch the HA log for the moment the "Home Assistant has restarted" toast appears at the bottom of the UI. That is `homeassistant: start` firing. From that point, watch how long it takes for entities to stop bouncing between states. Whatever that interval is on your system, that is the right window, with a small margin for headroom.

## Worked Example

The package YAML for the default case:

```yaml
template:
  - use_blueprint:
      path: TheThinkingHome/restarting_status.yaml
      input:
        restart_window_seconds: 60
```

This creates `binary_sensor.restarting_status`, which flips to `on` when `homeassistant: start` fires (typically up to a minute after boot, once every integration has finished loading), stays on for 60 seconds, then flips to `off`. Combined with the `unknown` state during the loading phase, dependents are blocked for roughly two minutes total wall-clock on a default setup.

A typical use in a downstream automation, as a condition that blocks the automation during boot:

```yaml
conditions:
  - alias: Not during the HA restart window
    condition: state
    entity_id: binary_sensor.restarting_status
    state: 'off'
```

A waiter that holds an automation until the system has finished settling, with a timeout so the automation does not hang forever if something goes wrong:

```yaml
- alias: Wait for HA to finish restarting
  wait_for_trigger:
    - trigger: state
      entity_id: binary_sensor.restarting_status
      to: 'off'
  timeout: "00:03:00"
  continue_on_timeout: true
```

A trigger that runs an automation precisely when the restart window closes (useful for sending a "system back up" notification, or for re-running a scene that should reflect the post-boot state of the house):

```yaml
triggers:
  - trigger: state
    entity_id: binary_sensor.restarting_status
    to: 'off'
```

## Parameters

| Input | Required | Default | Accepted values | What it does |
| --- | --- | --- | --- | --- |
| `restart_window_seconds` | No | `60` | integer between 15 and 300 | How many seconds after `homeassistant: start` fires that the system is considered to be still settling. The binary sensor is `on` during this window and `off` afterwards. Default 60 seconds covers residual settling on most systems (Z2M republish, MQTT reconnects, presence sensor wakeup). Raise for slower systems, lower for faster ones. |

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>)

This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.

## Changelog

| Version | Notes |
| --- | --- |
| 1.0.0 | Initial release. Trigger-based template binary_sensor fires once per HA boot from the `homeassistant: start` event (which HA fires when it transitions from "starting" to "running", after every integration has finished setup). State is a literal true; `auto_off` flips the sensor off after the configured window elapses. |
