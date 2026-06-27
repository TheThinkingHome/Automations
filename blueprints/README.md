# The Thinking Home Blueprints

A collection of Home Assistant blueprints from [xeazy.com](https://xeazy.com). Each one is capable, and built to do a real job well rather than to be trivial. They have settings worth understanding, so each blueprint's own page is worth reading: these are not click-import-and-forget. They come in two kinds: **automation blueprints** are ready to use right after import, while **template blueprints** need one small `use_blueprint` block in your configuration to become a usable entity. The two sections below are listed separately for that reason. Questions and discussion live on the forum: https://xeazy.com/logbook

## Automation Blueprints

Blueprints that watch for events and run actions in response. After import, click Create Automation on the Blueprints page, fill in the inputs, save, and the automation is running.

### Linked Entities Pro

Two switches that control the same fixture have to stay in sync, or the next press lands wrong. The wall switch and the smart bulb it controls, the Zigbee switch and the Tasmota module wired to a pair of table lamps, an `input_boolean` standing in for a physical control: the moment they disagree, the press goes the wrong way, the switch says the light is on but it is not, and the dashboard misleads. **Linked Entities Pro** keeps them aligned. Any number of entities, any mix of integrations, in one linked group, so a change on any of them propagates to the others. 

**[Full directions and one-click import ->](automation/linked_entities_pro.md)**

### Sensor Watchdog

Some smart devices fail silently. An Aqara FP2 presence sensor locks up at its last reading, the entity sits there forever pretending everything is fine, and the only fix is a power cycle. **Sensor Watchdog** automates that with the addition of a smart plug. Point it at one or more monitored entities, a recovery switch, and a timer helper. It cycles the plug when a monitored entity goes `unavailable`, when they stop reporting fresh values, or on a preventive reboot schedule. Pair multiple entities from a single device to get better coverage: an FP2's presence and illuminance entities cover complementary parts of the day, and reports from either count as a heartbeat.

**[Full directions and one-click import ->](automation/sensor_watchdog.md)**

### Sentinel Notify

A [Battery Sentinel](template/battery_sentinel.md) or [Entity Sentinel](template/entity_sentinel.md) sensor knows what is wrong, but it does not tell you, that is by design, each is a signal, not a notifier. **Sentinel Notify** is the companion built specifically to be their mouthpiece. Point it at one or more Sentinel sensors and it turns their output into a notification: the low batteries by name, level, and area, or the entities that have gone quiet, sent to any number of mobile devices. It notifies when there is a change in the monitored entites rather than on every cycle, so a battery that stays low does not nag you over and over. And a separate alert fires if a source Sentinel itself stops working, so a broken monitor reports itself instead of failing silently.

**[Full directions and one-click import ->](automation/sentinel_notify.md)**

## Template Blueprints

Blueprints that build derived sensors and helpers from your existing entities. After import, these do not get a Create button. You build entities from them by adding a short `use_blueprint` block under your `template:` configuration. The full walkthrough is [below the lists](#turning-a-template-blueprint-into-an-entity).

### Battery Sentinel

A house full of battery devices means a steady trickle of dying batteries, and the ones you forget are the ones that fail when you need them. **Battery Sentinel** watches the battery devices you choose and reports, as a sensor, which are running low against your chosen threshold. Its state is the count of devices with low batteries and its attributes list each low device by name, level, and area. It is the sister of [Entity Sentinel](template/entity_sentinel.md): this one answers "what is running low," the other answers "what stopped reporting." Build either or both, and pair them with [Sentinel Notify](automation/sentinel_notify.md) to turn their signals into alerts.

**[Full directions and one-click import ->](template/battery_sentinel.md)**

### Entity Sentinel

Some devices do not go offline cleanly. They lock up at their last reading, and the entity sits there showing a healthy value while the device behind it has gone quiet. A scheduled whole-house scan that only looks for `unavailable` never sees it. **Entity Sentinel** watches the specific entities you choose and reports, as a sensor, any that have stopped reporting, both the ones marked `unavailable` and the ones frozen at a stale value. Its state is the count of stale or unavailable entites and its attributes list exactly which entities are quiet, so a dashboard, an automation, or its notification companion can read the same signal. It is the sister of [Battery Sentinel](template/battery_sentinel.md): this one answers "what stopped reporting," the other answers "what is running low." Pair either with [Sentinel Notify](automation/sentinel_notify.md) to turn the signal into alerts.

**[Full directions and one-click import ->](template/entity_sentinel.md)**

### Recently Active

Home Assistant triggers fire once, at the instant a state crosses a line, and then the moment is gone. **Recently Active** holds onto it. It builds a binary sensor that stays `on` while your source is on, and for a chosen number of seconds after it turns off, so a fired-and-gone event becomes a state your other automations can still ask about. Point it at a door contact, a motion sensor, a switch, anything that reads on or off. The usual job is keeping a light that is controled by a motion sensor on, suppressing a notification for a short window right after something happens, such as skipping a camera's "a person is at the front door" alert after the door was opened.

**[Full directions and one-click import ->](template/recently_active.md)**

### Sensor Failover

A sensor going `unavailable` takes down whatever depends on it: an automation loses its reading and does the wrong thing, or simply stops working. **Sensor Failover** is a wrapper that reports your sensor's value while it is working, and the moment it goes unavailable, falls back to a backup sensor instead of going unavailable itself. You give it a primary and a list of fallbacks, point your automation at the wrapper, and the reading keeps coming even when the primary drops out.

**[Full directions and one-click import ->](template/sensor_failover.md)**

### System Stability

After every Home Assistant restart there is a window where things are settling: integrations stumble out of bed, Zigbee devices republish their state, presence sensors blink wildly trying to remember what a human looks like. Automations that fire on those state changes are firing on noise, not on real events. **System Stability** builds a binary sensor that reads `off` during shutdown and through the settling window after each restart, then `on` once the system is fully stable. Add `state == 'on'` as a condition to any automation with a misfire risk during the boot window, and the misfire absorbs cleanly. The usual job is gating motion-light automations, scene dispatchers, security notifications, and any other automation that misfires on the boot republish.

**[Full directions and one-click import ->](template/system_stability.md)**

### Weighted Confidence

Some states are not one sensor's job. Whether the house is emptly, whether a room is really in use, whether everyone has actually settled in for the night: each is a handful of weaker signals that mostly agree. **Weighted Confidence** takes a list of those signals, gives each a weight, and turns a binary sensor on when the agreeing weights cross a threshold. Signals that go unavailable drop out of the sum instead of dragging the score down, any signal can be made a hard gate that forces the answer off on its own, and the attributes show you the running score and exactly which signals are helping and which are holding it back. It pairs with Recently Active: feed in a "door shut for ten minutes" sensor as one of the weighted signals.

**[Full directions and one-click import ->](template/weighted_confidence.md)**

---

Everything below is the general guide for importing and using the blueprints here. The act of importing is the same for both kinds, but what happens after diverges: an automation blueprint behaves like any other automation right away, while a template blueprint lands on disk and needs one extra step to become a working entity. Each blueprint's own page, linked above, adds the specifics unique to it.

## How the blueprints are organized

They live under `blueprints/`, sorted by the kind of entity they build: `automation/` for automations, `template/` for template entities such as sensors and binary sensors, with `script/` to follow as the collection grows. Home Assistant works out which kind a blueprint is from inside the file, so the folder is only for tidiness.

## Importing a blueprint

Every blueprint page has a one-click import button. To do it by hand instead:

1. In Home Assistant, open Settings, then Automations & Scenes, then the Blueprints tab.
2. Click Import Blueprint at the bottom right.
3. Paste the blueprint's raw URL, listed on its page, and click Preview.
4. Confirm and import.

Importing only registers the blueprint, and what you see afterward depends on the kind. An automation blueprint now appears in the Blueprints tab. A template blueprint does not show up there at all; it lands on disk instead, and you build an entity from it with one more step described below.

**Automation blueprints** appear on the Blueprints page with a Create Automation button beside them. Click it, fill in the inputs the blueprint asks for, save, and the automation is running. That is the whole flow.

**Template blueprints** do not appear in the Blueprints list at all, and they never show up in the Create Helper list either. They land on disk at `config/blueprints/template/<author>/<name>.yaml`, and that file is what runs. Open that folder and note the exact `<author>` subfolder Home Assistant assigned, because you point at it when you build an entity. The next section walks through that.

## Turning a template blueprint into an entity

This is the step that catches people. Because a template blueprint never appears in the Blueprints list and gets no Create button, there is no UI flow to turn it into an entity. You do it with a short piece of YAML called `use_blueprint`:

```yaml
use_blueprint:
  path: <author>/<name>.yaml
  input:
    # the settings this blueprint asks for, listed on its page
```

`path` is relative to `config/blueprints/template/`, so it is the `<author>` folder plus the filename. The `input:` keys change from one blueprint to the next, and each blueprint's page spells out its own.

That `use_blueprint` block has to sit under Home Assistant's `template:` configuration. The tidy way to do that, and the way that holds up once you have several, is a package.

## What a package is, and how to use one

A package is a single YAML file that holds a bundle of related configuration. Rather than scattering entities across your main files, you keep everything for one purpose in one file, and Home Assistant folds it into your overall configuration at startup. Packages are optional, but they keep related things together and easy to find.

If you have never used packages, switch them on once. In `configuration.yaml`, add the two lines below. If a `homeassistant:` section already exists, just add the `packages:` line under it:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Then make a folder named `packages` next to your `configuration.yaml`. Every file you drop in there is a package.

Now create a file such as `packages/blueprints.yaml` and gather your `use_blueprint` blocks under one `template:` list:

```yaml
template:
  - use_blueprint:
      path: <author>/<name>.yaml
      input:
        # this entity's settings

  - use_blueprint:
      path: <author>/<another>.yaml
      input:
        # another entity's settings
```

Any mix of blueprints can share that list. One that builds a binary sensor and one that builds a sensor sit side by side, because each blueprint decides what kind of entity it makes.

If you already keep template entities in `configuration.yaml` under a `template:` key, you can drop a `use_blueprint` block into that list instead. The package is simply the cleaner home once you have a few.

## Loading your changes

A brand-new package file needs a full Home Assistant restart to register the first time. After that, adding more `use_blueprint` blocks to the same file only needs Developer Tools, YAML, Reload Template Entities.

## Updating a blueprint to a newer version

How you update depends on the kind, because the two kinds live in different places.

**Template blueprints** update by importing again. Go to the repository, find the import link for the blueprint you want to update, and click it. It overwrites the old version on your system with the new version from the repository. Then reload with Developer Tools, YAML, Reload Template Entities so the change takes effect.

**Automation blueprints** appear in the Blueprints list (Settings, Automations & Scenes, Blueprints). To update one, click the three dots beside it and choose Re-import Blueprint, which overwrites your copy with the current version. Then reload automations or restart Home Assistant.

One more thing worth knowing for either kind: just after a new version is published, the raw file can take a few minutes to update on GitHub's side, so a re-import that still pulls the old version right after a release is usually that delay. Wait a few minutes and try again rather than repeating it back to back.

## License

Copyright (C) 2026 James Lander, The Thinking Home (https://xeazy.com) This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.
