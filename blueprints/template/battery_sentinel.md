# Battery Sentinel (Beta)

A Home Assistant template blueprint that watches every battery device in the house and reports the ones running low, as a sensor anything in your system can read.

Battery monitoring in Home Assistant has looked the same for years. The common blueprints check each device's battery percentage on a schedule and push a notification to your phone. The notification is the entire product. You get a one-time alert and nothing else in your system can read the result.

The forums carry a steady run of feature requests that a notification-only design cannot reach: list the actual battery type so the alert tells you what to buy, filter selected battery devices with a `battery_watch` label, send the result somewhere other than a phone, surface the device that is currently offline rather than just low. These are not hard requests. They are simply out of reach for a blueprint whose only output is a notification.

**Battery Sentinel** takes a different starting point. Instead of building down from a notification, it builds up from a sensor.

Its companion, **Entity Sentinel**, watches any critical entity for going quiet, unavailable or frozen at its last value. Battery Sentinel answers "what is running low," Entity Sentinel answers "what stopped reporting." Build both for full coverage, or just the one you want. They are separate blueprints and do not depend on each other.

## Why a Sensor Instead of an Automation

Battery Sentinel produces a `sensor`, not a one-time action. A sensor has a state and attributes that persist, and anything in Home Assistant can read it. One sensor, many listeners:

- **A notification**, the obvious one. A companion automation blueprint for this is planned, so you get the polished alert without hand-writing the Jinja yourself.
- **A dashboard card** that lists the low devices with area and battery type, so a glance tells you what to grab from the drawer.
- **A to-do or shopping list**, each low cell dropped onto a list so the right battery is bought before the swap.
- **A voice summary** through Assist, asking which batteries need attention before you head out.
- **A gate on another automation**, logic that only runs when the list is empty or non-empty.

The notification is the least of what the signal can drive. It is just the first most people reach for.

## What It Does

Battery Sentinel counts the batteries at or below a threshold and lists each one, with its percentage, area, and battery type, in a `devices` attribute. The state is the count; the detail is in the attributes. You decide what reads it.

- **Percentage and binary, both handled.** A device counts as low when its battery percentage is at or below the threshold, or when a binary battery sensor reads `on`. A binary low battery sensor has no number to report, so its `level` comes through as `null`.
- **Hysteresis so it does not flap.** A value-hysteresis band keeps a device hovering at the threshold from bouncing in and out of the list. A flagged device stays low until it climbs above the threshold plus a margin, which for a battery only happens when a fresh cell is fitted.
- **Structured output.** The `devices` list holds objects carrying the device name, area, percentage level, and battery type if available. Every consumer can format them in its own way.
- **Offline batteries surfaced separately.** Any in-scope battery sitting at `unavailable` or `unknown`, an offline device, an integration that dropped an entity, a typo'd entity id, is not a low battery, so it never lands in the low list. Instead it is reported in its own `unavailable_entities` attribute, with a count in `unavailable_count`. The low state stays clean while you still get a window onto what is currently offline.
- **A grace period after a restart, measured from a reliable clock.** A momentary restart briefly reads most of the mesh as `unavailable` while it repopulates. For a window after Home Assistant starts, the unavailable list is held empty so a restart never reports a false fleet-wide outage. That window is measured from a required uptime sensor, which makes it reliable across restarts and reloads; see the next section. The low count is never held; it evaluates live the whole time.
- **A loud error if it is set up wrong.** If the uptime sensor is missing or misconfigured, or if a threshold or margin value is set to something invalid, the sensor reports a `setup_error` state and does nothing else until you fix it, rather than failing quietly. An `ok` attribute lets your automations tell a working sensor from a broken one.
- **A manual refresh button.** Point an optional `input_button` at the sensor and a press re-evaluates it on the spot, handy right after you change a battery instead of waiting for the next scan. It re-scans; it cannot make a device report a fresh reading it has not sent yet.
- **Scope by label, area, device, or entity**, with include and exclude, and exclude always wins. Area and device scopes are filtered to real battery entities; labels expand to every battery entity they cover, whether the label sits on the entity, its device, or its area.

This is a beta. It is an advanced blueprint: it requires the uptime integration in timestamp mode and a startup grace period you set to cover your own boot and mesh-settle time. Read the next two sections before you import.

Full write-up and worked examples: <https://xeazy.com/battery-entity-sentinel-blueprint/>  Questions and discussion: <https://xeazy.com/logbook/>

## Before You Start: the Uptime Requirement

Battery Sentinel needs one thing in place before it will work: a Home Assistant **uptime sensor in timestamp mode**.

The reason is the startup grace period. After a restart, the sensor needs to know how long the system has been up, so it can hold the unavailable list while the mesh comes back. The obvious way to track that, reading the sensor's own last boot time from its attributes, is unreliable: Home Assistant has a long-standing bug ([#115585](https://github.com/home-assistant/core/issues/115585)) where a trigger-based template sensor sometimes cannot read its own attributes during evaluation. Building the grace period on that foundation means it occasionally fails open and floods you with false alerts right after a restart, the exact moment it is supposed to protect.

The uptime sensor sidesteps the whole problem. It is an independent entity whose value, the timestamp the Home Assistant process started, is read fresh on every evaluation and never depends on the sensor's own attributes. It is reliable across restarts, reloads, and the bug.

Most systems already have the uptime sensor. If yours does not, add the **Uptime** integration (Settings, Devices & Services, Add Integration, Uptime). Then make sure the entity is in **timestamp** mode: its state should read like `2026-06-26T14:56:59+00:00`, an ISO timestamp, not a number of days. If it shows a number, change the entity's "displayed unit" or the integration option to timestamp.

If the uptime sensor is missing, unavailable, or not a timestamp, Battery Sentinel will not guess. It reports a `setup_error` state with a message telling you exactly what is wrong, watches nothing, and starts working the moment you fix it.

## Import the Blueprint

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Ftemplate%2Fbattery_sentinel.yaml)

Or paste this URL into Settings, Automations & Scenes, Blueprints, Import Blueprint:

```
https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/template/battery_sentinel.yaml
```

Importing only registers a blueprint. It does not create a sensor yet. After import, the file lands at `config/blueprints/template/TheThinkingHome/battery_sentinel.yaml`. Verify it is there, because you will need that path in the next step.

## Setting Up the Sensor

### The Catch with Template Blueprints

Automation blueprints get a friendly Create automation button right on the Blueprints page. Template blueprints do not. You will not find this in the Create Helper list, and that is expected, not a fault. Template blueprints become real entities through a short piece of YAML called `use_blueprint`. It takes a path to the blueprint and a set of inputs.

### What a Package Is, and How to Create One

A package is a single YAML file that holds a bundle of related configuration. Rather than scattering entities across your main files, you put everything for one feature into one file, and Home Assistant folds it into your overall configuration at startup. Packages are optional, but they keep related things together and easy to find.

If you have never used packages, switch them on once. In your `configuration.yaml`, add the two lines below. If you already have a `homeassistant:` section, just add the `packages:` line underneath it:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Then create a folder named `packages` next to your `configuration.yaml`. Every YAML file you drop in there is now a package.

Now make a package file, for example `packages/blueprint_battery_sentinel.yaml`, and put the `use_blueprint` block inside a `template:` list:

```yaml
template:
  - use_blueprint:
      path: TheThinkingHome/battery_sentinel.yaml
      input:
        low_threshold: 20
        low_clear_margin: 2
        startup_grace_seconds: 240
        uptime_sensor: sensor.uptime
        scan_interval: "/2"
        sensor_name: Battery Sentinel
        unique_id: battery_sentinel
```

To narrow the sensor, add an `include_target` with a label, area, device, or entity, and an `exclude_target` for anything to drop. If you already keep template entities in your `configuration.yaml` under a `template:` key, you can drop the `use_blueprint` block into that list instead. The package is simply the cleaner home once you have a few. If you also build Entity Sentinel, the natural choice is to put both `use_blueprint` blocks in the same package file, each with its own `unique_id`.

### Load It

A brand-new package file needs a full Home Assistant restart to register the first time. After that, adding more `use_blueprint` blocks to the same file only needs a quick reload through Developer Tools, YAML, Reload Template Entities.

When it comes up you will have a sensor named after `sensor_name`. Watch it in Developer Tools, States. The state is the count, and the `devices` attribute lists the flagged devices.

## Setting the Grace Period

The `startup_grace_seconds` value is the one input you should think about rather than accept blindly, because the right number depends on your hardware.

The grace period covers the gap between Home Assistant starting and your battery devices being fully back online. It is measured from the uptime sensor, which marks when the Home Assistant process started. So the window has to cover your hub's **full** startup: the boot itself, plus the time your Zigbee or Z-Wave mesh takes to repopulate, plus a margin.

The default is `240` (four minutes), which suits a fast mini-PC with a Zigbee mesh. Tune from there:

- A fast system (an N100 or better, Zigbee) is usually settled in two to three minutes. Four minutes gives headroom.
- A slower hub, or a large or Z-Wave mesh that is slow to repopulate, may need five or six minutes. If you see a burst of false `unavailable` reports in the first few minutes after every restart, the grace is too short; raise it.
- Set it to `0` to disable the grace entirely, only sensible if your devices come back effectively instantly.

The cost of setting it too high is small: for those extra minutes after a restart, a battery that is genuinely offline will not appear in the unavailable list yet. The low count is never affected; it works the entire time. So err on the generous side.

## Scoping: Choosing What the Sensor Watches

The sensor decides which battery devices to watch through two inputs, `include_target` and `exclude_target`. Each accepts entities, areas, devices, or labels, and you can mix them.

### The Default: Watch Everything

Leave `include_target` out entirely and the sensor watches every battery device-class entity in your system. This is the widest scope and a fine place to start. On most systems it also catches phones, tablets, and watches, which carry battery entities through the companion app. Those charge nightly and are rarely what you want in a low-battery list, which is what `exclude_target` is for.

### Include by Entity

Name the exact entities you want, and the sensor watches precisely those. An explicit entity list is trusted: the blueprint watches what you named and does not second-guess it.

```yaml
        include_target:
          entity_id:
            - sensor.motion_master_battery
            - sensor.leak_kitchen_sink_battery
            - binary_sensor.door_garage_battery_low
```

### Exclude, and Exclude Always Wins

`exclude_target` takes the same kinds of targets and removes them from whatever the include produced. Exclude beats include every time, so it is how you drop the always-charged devices from a wide scope:

```yaml
        exclude_target:
          entity_id:
            - sensor.phone_james_battery_level
            - sensor.tablet_kitchen_battery_level
```

### Filter by Area or Device

Point the scope at an area or a device, and the sensor watches the battery entities within it. Areas and devices are **swept**: an area holds every entity assigned to it, and a device exposes many entities beyond its battery, so the blueprint keeps only the `device_class: battery` entities and drops the rest (signal strength, connectivity, blinds position, and so on).

```yaml
        include_target:
          area_id:
            - master
            - master_bath
```

### Filter by Label

A label lets you tag the exact devices you care about and watch only those, for example a `battery_watch` label on your critical infrastructure.

Home Assistant lets you apply a label to an entity, a device, or an area, and these are separate attachment points. Battery Sentinel reads **all three**. A label applied directly to a battery entity is found. A label applied to a *device* is expanded to that device's battery entities. A label applied to an *area* is expanded to the battery entities in that area. So you can label whatever is most convenient, the entity, its device, or its room, and the sensor finds the battery either way.

```yaml
        include_target:
          label_id:
            - battery_watch
```

This expansion works on both sides. On the include side it is filtered to battery entities, so labeling a device pulls in only its battery, not its other entities. On the exclude side it is expanded fully, so a label used to suppress a device reliably removes that device's battery from the list. A device labeled to be ignored is actually ignored, not silently watched anyway.

## What the Sensor Looks Like

With two cells below the threshold and one battery currently offline:

```yaml
sensor.battery_sentinel
State: 2

ok: true
error: ''
uptime_status: '2026-06-24T09:40:00-05:00'
total_monitored: 27
unavailable_count: 1
unavailable_entities:
  - name: Hall Motion
    entity_id: sensor.hall_motion_battery
    area: Hall
    state: unavailable
last_evaluated: '2026-06-24T10:15:00-05:00'
devices:
  - name: Master Bath Motion
    entity_id: sensor.master_bath_motion_battery
    area: Master Bath
    level: 12
    battery_type: CR2450
  - name: Back Door
    entity_id: binary_sensor.back_door_battery
    area: Back Door
    level: null
    battery_type: ''
```

The state is the single number you badge or trigger on, or the string `setup_error` if the sensor is misconfigured. The `devices` list is there for any consumer that wants the detail. `total_monitored` reports how many battery entities fell within the scope. A binary device shows `level: null`, since "on" carries no percentage. `unavailable_entities` is a separate list of in-scope batteries currently `unavailable` or `unknown`, each with its raw `state` (`unavailable`, `unknown`, or `missing` for an entity id that resolves to nothing), and `unavailable_count` is its length. These never affect the low state; they are a parallel view of what is offline.

### Attribute Reference

- **`state`** the number of batteries currently low, or the string `setup_error` if the sensor is misconfigured (see below). Zero means everything in scope is above the threshold and reporting.
- **`ok`** a boolean, `true` when the sensor is functioning (whether or not it has flagged any low batteries), `false` only when it is in `setup_error`. This is the safe field to gate downstream automations on; see the warning under "Putting the Signal to Work."
- **`error`** empty in normal operation. When the state is `setup_error`, this names the input that is wrong and what to do about it.
- **`uptime_status`** the parsed uptime timestamp in normal operation, or the raw bad value when the uptime sensor is the problem. A diagnostic aid.
- **`total_monitored`** how many battery entities fell within the scope.
- **`devices`** the list of low batteries, each with `name`, `entity_id`, `area`, `level` (null for binary), and `battery_type` if the device exposes it.
- **`unavailable_count`** / **`unavailable_entities`** the parallel list of in-scope batteries currently `unavailable`, `unknown`, or `missing`, held empty during the startup grace.
- **`boot_time`** the start time the grace is measured from, or null in `setup_error`.
- **`settled`** false during the startup grace window, true afterward.

### The `setup_error` State

If the sensor cannot run correctly, it does not fail quietly or report a misleading zero. It sets its state to the string `setup_error`, sets `ok` to false, and puts a plain-language explanation in the `error` attribute. It watches nothing and counts nothing until the problem is fixed, then recovers on its own at the next evaluation.

Two kinds of problem trigger it. The first is the uptime sensor: missing, unavailable, or not in timestamp mode. The second is an input set to an invalid value:

- `low_threshold` must be a number between 0 and 100 (it is a battery percentage). A value like 150 or -5 is rejected.
- `low_clear_margin` must be a number of 0 or greater. A negative margin would invert the hysteresis band.
- `startup_grace_seconds` must be a number of 0 or greater.

The import UI constrains these for you, so you will mostly meet `setup_error` only if you deploy through a `use_blueprint` package and mistype a value, or if the uptime sensor is not set up. The error message names the specific input and the fix.

## Parameters

| Input | Required | Default | Accepted values | What it does |
| --- | --- | --- | --- | --- |
| `unique_id` | Yes | none | any short text id, distinct per sensor | Registers the entity so you can rename it or place it in an area. Reusing one collides two sensors. |
| `uptime_sensor` | Yes | `sensor.uptime` | an uptime sensor in timestamp mode | The clock the startup grace is measured from. Required. Missing or non-timestamp yields `setup_error`. See "Before You Start." |
| `low_threshold` | No | `20` | 0 to 100 | A battery percentage at or below this counts as low. Binary battery sensors reading `on` always count as low. Out of range yields `setup_error`. |
| `low_clear_margin` | No | `2` | 0 or greater | How far above the threshold a flagged device must climb before it clears. The hysteresis band that stops flapping. Negative yields `setup_error`. |
| `include_target` | No | empty | entities, areas, devices, or labels | Which devices to watch. Empty watches every battery device-class entity. Areas and devices are filtered to batteries; labels expand to the batteries they cover on entity, device, or area. |
| `exclude_target` | No | empty | entities, areas, devices, or labels | Which to leave out. Exclude always wins over include. |
| `scan_interval` | No | `/2` | `/1`, `/2`, `/3`, `/5`, `/10`, `/15`, `/30` | How often the sensor re-scans, in hours. |
| `startup_grace_seconds` | No | `240` | 0 or greater (seconds) | After Home Assistant starts, how long to hold the `unavailable_entities` list empty while the mesh repopulates, measured from the uptime sensor. The low count is never held. Set to 0 to disable. See "Setting the Grace Period." |
| `refresh_button` | No | none | an `input_button` entity | Optional. Press the named button to re-evaluate the sensor immediately, for example after changing a battery. It re-scans; it does not force a device to report. Several sensors can share one button. Leave unset to disable. |
| `sensor_name` | No | `Battery Sentinel` | any text | The friendly name, and what the entity id is built from. |
| `debug_enabled` | No | `false` | on or off | When on, writes one diagnostic line to the system log each evaluation, carrying the version, state, and both the low and unavailable lists. |

## A Note on Hysteresis After a Restart

The hysteresis band, the rule that keeps a flagged battery low until it climbs past the threshold plus the margin, remembers which batteries were low on the previous evaluation. That memory is the one piece of state the sensor reads from its own prior output, because "was this battery low last time" is inherently self-referential; there is no external source for it the way there is for the startup clock.

In the rare event that Home Assistant's [#115585](https://github.com/home-assistant/core/issues/115585) bug strikes on a given evaluation, that memory reads empty for that one tick, and a battery sitting inside the clear margin (say 21 percent with a threshold of 20 and a margin of 2) briefly drops off the low list, returning on the next clean evaluation. This is a known, self-correcting limitation. It cannot be engineered away without giving up hysteresis altogether, and a one-cycle flicker is a fair price for a deadband that otherwise keeps a marginal battery from bouncing in and out of your alerts.

## Putting the Signal to Work

The sensor is the foundation. Here are a few things to build on it.

A dashboard card listing what is low, with the battery type so you know what to fetch:

```yaml
type: markdown
content: >
  {% set s = state_attr('sensor.battery_sentinel', 'devices') %}
  {% if s %}**Low batteries:**
  {% for d in s %}
  - {{ d.name }}{% if d.level is not none %} ({{ d.level }}%){% endif %}{% if d.battery_type %} - {{ d.battery_type }}{% endif %}
  {% endfor %}
  {% else %}All batteries above threshold.{% endif %}
```

A gate that holds back an automation when the battery list is not clean. Note the `ok` check first: it confirms the sensor itself is healthy before trusting its count, so a misconfigured sensor does not wave the automation through:

```yaml
condition:
  - condition: template
    value_template: >-
      {{ is_state_attr('sensor.battery_sentinel', 'ok', true)
         and states('sensor.battery_sentinel') | int(0) == 0 }}
```

**A word on the state and downstream math.** In normal operation the state is a number, but if the sensor is misconfigured the state becomes the string `setup_error`. Do not gate automations on `int()` math alone: `{{ states('sensor.battery_sentinel') | int(0) == 0 }}` reads `setup_error` as 0 and would treat a broken sensor as "no low batteries, all clear." Check the `ok` attribute first (`is_state_attr('sensor.battery_sentinel', 'ok', true)`), which is true only when the sensor is actually working, then read the count.

A daily summary through a notification, fired by your own automation reading the `devices` attribute, so the alert names the device, the area, and the battery to buy.

## Changelog

| Version | Notes |
| --- | --- |
| 1.0.0-beta | First beta. Counts batteries at or below a threshold with a hysteresis deadband, handles percentage and binary battery entities, reports unavailable and missing batteries in a parallel `unavailable_entities` attribute. Scope by entity, label, area, or device with include and exclude (exclude wins; labels expand to entities, devices, and areas on both sides, so a device-labeled exclude actually suppresses its battery). Startup grace measured from a required uptime sensor in timestamp mode, reliable across restarts and reloads. Loud `setup_error` state with `error` and `uptime_status` attributes when the uptime sensor is missing or not a timestamp, or when `low_threshold`, `low_clear_margin`, or `startup_grace_seconds` is set to an invalid value; the sensor fails safe and self-heals. Added an `ok` boolean so downstream automations can gate on it rather than doing integer math on a state that may read `setup_error`. Timezone-safe grace math. Debug log line carries the version, state, uptime status, and both the low and unavailable arrays. |

## License

GPL-3.0-or-later. Copyright (C) 2026 James Lander, The Thinking Home ([xeazy.com](https://xeazy.com)).

See the full YAML blueprint, the instructions, and the one-click import button at the [Thinking Home blueprints repository](https://github.com/TheThinkingHome/Automations).
