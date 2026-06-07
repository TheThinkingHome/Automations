# Recently Active

A Home Assistant template blueprint that turns a fired-and-forgotten event into a state you can still ask about a little while later.

Home Assistant triggers fire once, at the instant a state crosses a line. The door opens, the event fires, and it is gone. Ask a moment later whether the door was just opened and there is nothing left to read, because Home Assistant evaluated that crossing once and moved on. Recently Active closes the gap. It builds a binary sensor that reads `on` while your source is on, and keeps reading `on` for a chosen number of seconds after the source turns off. So "the door is open" quietly becomes "the door was open within the last N seconds," which is the question your other automations actually want to ask.

It is not limited to on/off sources. Point it at a numeric sensor and it reads `on` while the value is above a threshold, or while it is below one, with the same linger. That covers the dishwasher that pauses mid-cycle and the reading that dips across the line for a moment: the sensor holds steady instead of flapping on every brief crossing.

Full write-up and a worked example: https://xeazy.com/how-to-use-a-home-assistant-blueprint-template-sensor-the-recently-active-sensor/
Questions and discussion: https://xeazy.com/logbook/d/36-recently-active-blueprint

## What you get

- A `binary_sensor` that reads `on` while the condition holds, and for a set number of seconds after it clears.
- Three ways to set the condition: an on/off source is on, a numeric sensor is above a threshold, or a numeric sensor is below a threshold.
- Works with any on/off source (a `binary_sensor`, a `switch`, an `input_boolean`) or any numeric `sensor`.
- If the source goes unavailable or unknown, the sensor returns `off`, so a dead source never sticks the sensor on and never silently suppresses whatever depends on it.

How the timing works, up front: the linger runs on the binary sensor's own `delay_off`, so the off transition lands exactly when your window closes, to the second, and the condition clearing is caught instantly. What it does not give you is a timer you can pause, cancel, or watch count down. If you need any of that, a timer helper or History Stats is the better tool. This one is for clean, short linger in a single line.

## Requirements

Home Assistant 2026.5.4 or newer. That is the version it is verified on.

## Step 1: Import the blueprint

The quick way, one click:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Ftemplate%2Frecently_active.yaml)

Or by hand:

1. Go to Settings, then Automations & Scenes, then the Blueprints tab.
2. Click Import Blueprint at the bottom right.
3. Paste this URL and click Preview:
   ```
   https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/template/recently_active.yaml
   ```
4. Confirm and import.

Importing only registers the blueprint. It does not create a sensor yet. After import, the file lands at `config/blueprints/template/<author>/recently_active.yaml`, typically `TheThinkingHome/recently_active.yaml`. Open that folder and note the exact `<author>` subfolder name Home Assistant assigned, because you need it in the next step.

## Step 2: The catch with template blueprints

Automation blueprints get a friendly "Create automation" button right on the Blueprints page. Template blueprints do not. You will not find Recently Active anywhere in the Create Helper list, and that is expected, not a fault. Template blueprints become real entities through a short piece of YAML called `use_blueprint`. It is a path, your settings, and that is it:

```yaml
use_blueprint:
  path: TheThinkingHome/recently_active.yaml
  input:
    source_entity: binary_sensor.front_door
    unique_id: front_door_recently_open
    linger_seconds: 60
    sensor_name: Front Door Recently Open
    device_class: occupancy
```

`path` is relative to `config/blueprints/template/`, so it is the `<author>` folder from Step 1 plus the filename. `source_entity` and `unique_id` are the two required settings; the rest have sensible defaults. For a numeric source, set `comparison` to `above` or `below` and give a `threshold`. Every parameter is laid out in the reference below.

That block has to sit under Home Assistant's `template:` configuration. The tidy way to do that, and the way that scales once you make more of these, is a package.

## Step 3: What a package is, and how to use one

A package is a single YAML file that holds a bundle of related configuration. Rather than scattering a sensor here and an automation there across your main files, you put everything for one feature into one file, and Home Assistant folds it into your overall configuration at startup. Packages are optional, but they keep related things together and easy to find.

If you have never used packages, switch them on once. In your `configuration.yaml`, add the two lines below. If you already have a `homeassistant:` section, just add the `packages:` line underneath it:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Then create a folder named `packages` next to your `configuration.yaml`. Every YAML file you drop in there is now a package.

Now make a package file, for example `packages/recently_active.yaml`, and put your `use_blueprint` block inside a `template:` list:

```yaml
template:
  - use_blueprint:
      path: TheThinkingHome/recently_active.yaml
      input:
        source_entity: binary_sensor.front_door
        unique_id: front_door_recently_open
        linger_seconds: 60
        sensor_name: Front Door Recently Open
        device_class: occupancy
```

Each entity you build from the blueprint is one more list item in the same file. They can watch different sources, in different modes, with different windows, all in one place. Give each one its own `unique_id`, since reusing one collides the two sensors. Here is one of each mode in a single file:

```yaml
template:
  # on/off source
  - use_blueprint:
      path: TheThinkingHome/recently_active.yaml
      input:
        source_entity: binary_sensor.front_door
        unique_id: front_door_recently_open
        linger_seconds: 300
        sensor_name: Front Door Recently Used
        device_class: occupancy

  # numeric, above a threshold
  - use_blueprint:
      path: TheThinkingHome/recently_active.yaml
      input:
        source_entity: sensor.dishwasher_power
        comparison: above
        threshold: 3
        linger_seconds: 300
        sensor_name: Dishwasher Recently Running
        unique_id: dishwasher_recently_running
        device_class: running

  # numeric, below a threshold
  - use_blueprint:
      path: TheThinkingHome/recently_active.yaml
      input:
        source_entity: sensor.living_room_illuminance
        comparison: below
        threshold: 10
        linger_seconds: 120
        sensor_name: Living Room Recently Dark
        unique_id: living_room_recently_dark
```

If you already keep template entities in your `configuration.yaml` under a `template:` key, you can drop the `use_blueprint` block into that list instead. The package is simply the cleaner home once you have a few.

## Step 4: Load it

A brand-new package file needs a full Home Assistant restart to register the first time. After that, adding more `use_blueprint` blocks to the same file only needs a quick reload through Developer Tools, YAML, Reload Template Entities.

When it comes up you will have a `binary_sensor` named after your `sensor_name`. Watch it in Developer Tools, States: flip the source on, or push the value past the threshold, and it goes on; clear the condition and it holds on for your window, then drops to off.

## A small note on the initial state

Home Assistant's `delay_off` on a template binary_sensor has one bit of startup behavior that this blueprint inherits. After the entity is first created, and again after every Home Assistant restart, it starts at `unknown`. If your condition is true at that moment, the sensor immediately reads `on` and you would never notice. If your condition is false at that moment, the sensor may stay at `unknown` until the condition becomes true at least once. Home Assistant cannot tell whether a false result is a fresh transition (where `delay_off` should still be holding the entity `on`) or the actual initial state of a freshly created entity, so it picks the safer reading and waits.

The first time the condition does become true, the sensor goes `on`, and from that point forward `delay_off` is warm. When the condition clears, the linger counts down and the sensor lands cleanly on `off`. Every cycle after that works exactly as the rest of this README describes. Restarts reset the entity to `unknown` and the same warming behavior applies: if the condition is true at restart, the sensor jumps straight to `on`; if false, it waits for the first true.

In day-to-day use you do not notice this. An on/off source like a door contact or a motion sensor hits `on` within minutes of any restart, and the unknown window closes itself. The case that surprises people is a numeric source whose value sits on one side of the threshold for a long stretch. Outdoor illuminance overnight, a freezer holding cold, an idle appliance's power meter. The condition can stay false for hours, the sensor may read `unknown` for that whole stretch, and it will resolve the next time the value crosses.

To confirm a new sensor works without waiting for natural conditions to flip, you can set the source's state from Developer Tools, States to a value that satisfies the condition, watch the sensor go `on`, and set it back. One catch: this fires real state-change events on the source, so anything else watching it will react.

For the record, this is documented behavior of Home Assistant's template binary_sensor with `delay_off`, not a quirk of this blueprint. The relevant threads are core issue [#64423](https://github.com/home-assistant/core/issues/64423), core issue [#66376](https://github.com/home-assistant/core/issues/66376), core issue [#67397](https://github.com/home-assistant/core/issues/67397), and community thread [#411956](https://community.home-assistant.io/t/sensor-with-delay-off-has-state-unknown-at-startup-when-off/411956).

Building on `delay_off` is a deliberate choice. It is lightweight, accurate to the second, and the state that automations care about is the on-to-off transition when the linger expires, not the initial reading. That transition only ever happens after the sensor has been on at least once, which means the unknown window at startup does not affect anything you are actually triggering on. Precision and simplicity in the steady state are worth the cosmetic cost on the very first reading.

## Where this fits

There are three reasonable ways to ask "is this still recently true?" in Home Assistant, and each has a sweet spot.

A trigger with `for:` is the simplest. The trigger only fires once the source has held its state for the full duration. No extra entities, no helpers, no template plumbing. It is the right tool when the linger is short and lives inside a single automation. Think 30 seconds or less, where you do not need the held state to be visible anywhere else and you do not need it to survive a Home Assistant restart.

A `timer` helper with two automations, one to start it and one to act when it ends, is the right tool when the linger is long and needs to persist across a restart, or when you want to pause, cancel, or extend it from the interface. Timers survive a Home Assistant restart, can be paused and resumed, and show their remaining time on a card. They cost a helper and a couple of automations, but for long-running windows of an hour or more, that cost is fair.

This blueprint sits in between. It builds a single entity that other automations can read directly as a condition, runs on the binary sensor's own `delay_off`, is accurate to the second, is lightweight enough to make several of without thinking about it, and reads like one block of YAML once the package is in place. For windows between roughly 30 seconds and an hour, especially when several automations need to read the same "recently true" state, it is the cleanest fit.

## Parameters

| Input | Required | Default | Accepted values | What it does |
|---|---|---|---|---|
| `source_entity` | Yes | none | any on/off entity (`binary_sensor`, `switch`, `input_boolean`), or any numeric `sensor` for the above/below modes | The entity to track. |
| `comparison` | No | `on` | `on`, `above`, `below` | How to read the source: `on` for an on/off source, `above` for a numeric value over the threshold, `below` for one under it. |
| `threshold` | No | `0` | any number | The value the source is compared against in the `above`/`below` modes. Ignored in `on` mode. |
| `unique_id` | Yes | none | any short text id, distinct per sensor | Registers the entity so you can rename it, place it in an area, or change its device class from the interface. Reusing one collides two sensors. |
| `linger_seconds` | No | `60` | a whole number of seconds, 1 to 3600 | How long the sensor stays on after the condition clears. |
| `sensor_name` | No | `Recently Active` | any text | The friendly name, and what the entity id is built from. |
| `device_class` | No | `occupancy` | any binary_sensor device class (`occupancy`, `motion`, `door`, and so on), or left blank | How the sensor reads in the frontend. `occupancy` shows Detected and Clear; blank shows plain On and Off. |

## A small note on display

By default the sensor uses the `occupancy` device class, so the frontend shows Detected and Clear rather than plain On and Off. Change the device class input to relabel it, for example `running` for Running and Not running, or clear the field for a plain On and Off. Pick the class for what the new sensor means, not for what its source is: a "recently used" door is really a presence signal, so `occupancy` reads truer than `door`, which would show Open while the door has actually been shut for the whole window. The underlying state value is `on` or `off` whichever you pick, so it reads correctly in templates and conditions regardless.

## Example use

A common one: an LLM analyzes snapshots from a front-door camera to flag unusual activity, but every ordinary entry trips the camera and the model dutifully reports a person walking through the door, which is noise. Point Recently Active at the door contact, and use its `on` state to suppress the analysis for a short window after the door is used. The full worked version is in the article linked at the top.

A numeric one: appliances draw power in bursts, so a plain "is it drawing power" check reads off during a lull and fires "cycle complete" mid-wash. Point Recently Active at the power sensor with `comparison: above`, a `threshold` just over the machine's idle draw, and a `linger_seconds` long enough to cover the longest quiet stretch. The sensor stays on through the pauses and only drops once the machine has truly stopped.

## License

Copyright (C) 2026 James Lander, The Thinking Home (https://xeazy.com)
This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.
