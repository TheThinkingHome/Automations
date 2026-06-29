# Battery Sentinel (Beta)

A Home Assistant template blueprint that watches the battery devices you choose and reports the ones running low. It is a sensor anything in your system can read: an automation, a voice assistant, a notification, a dashboard card.

## What This Solves

Battery monitoring in Home Assistant has looked the same for years. The common blueprints check every device's battery percentage on a schedule and push a notification to your phone. The notification is the entire product: a one-time alert where nothing else in your system can read the result.

The forums carry a steady run of requests a notification-only design cannot reach: listing the actual battery type so the alert tells you what to buy, creating a list of devices with a label, sending the result somewhere other than a phone, detecting devices that are currently offline rather than only the ones that are low. None of these are hard. They are simply out of reach for a blueprint whose only output is a notification.

Battery Sentinel starts with a different perspective. Instead of building down from a notification, it builds up from a sensor.

## Built on Request, and Why it is a Beta

Every other blueprint from _The Thinking Home_ series grew out of my own house. The sensors and automations behind them ran on my system for years before I shared them, so the reliability was already settled. My job generalized what already worked: made it configurable, documented it, smoothed the edges, and handed over something I already trusted.

This one is different. It was not refined over years while it ran quietly within my walls. It was asked for, by the community, for a problem the existing tools never solved. So I built it: imagined, drafted, edited, reimagined, debugged, and tested as hard as one house and an adversarial test suite allowed. I believe in it, but I cannot yet claim what years of uneventful running will allow me to claim.

That is why this is a beta release. It is genuinely useful today, and I run it on my own system. But if you use it, you are also helping test and refine it. Real homes are more varied than any one setup, and the edge cases that matter most are the ones I haven't thought of and cannot produce. If you hit something, the [community thread](https://xeazy.com/logbook/d/42-the-battery-entity-sentinel-blueprints) is where it gets found and fixed. That is the deal: I have done my best to build it, and the community's use is what will make it solid.

## Why a Sensor, Not an Automation

Most blueprints for this kind of job are automations: they run on a schedule, fire a notification, and that is the entire product. You get one alert and nothing in your system can read the result afterward.

This builds a `sensor` instead. A sensor has a state and attributes that persist, and anything in Home Assistant can read it. One sensor, many listeners:

- **A notification automation**, the obvious one. The [Sentinel Notify](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/automation/sentinel_notify.md) companion blueprint does exactly this, in this repository.
- **A dashboard card** that lists the low devices with area and battery type, so a glance tells you what to grab from the drawer.
- **A to-do or shopping list**, each low cell dropped onto a list so the right battery is bought before the swap.
- **A voice summary** through Assist, asking which batteries need attention before you head out.
- **A gate on another automation**, logic that only runs when the list is empty or not.

The notification is the least of what the signal can drive. It is just the first thing most people reach for. Building a sensor asks a little more setup than importing a fire-and-forget automation, but that small extra step is exactly what buys everything above: a result your whole system can read, reuse, and act on, instead of a single alert that vanishes after it fires.

## What It Does

Battery Sentinel counts the batteries at or below a threshold and lists each one, with its percentage, area, and battery type if available, in a `devices` attribute. The state is the count; the detail is in the attribute.

- **Percentage and binary, both handled.** A device counts as low when its battery percentage is at or below the threshold, or when a binary battery sensor reads `on`. 
- **Hysteresis so it does not flap.** A flagged device usually stays low. But it can fluctuate slightly as a load is removed when the device goes to sleep.
- **Offline batteries surfaced separately.** An in-scope battery sitting at `unavailable` or `unknown` is not a low battery, so it never lands in the low list. It is reported in a separate `unavailable_entities` attribute, so the low state stays clean while you still see what is offline.
- **A grace period after a restart.** For a window after Home Assistant starts, the unavailable list is held empty so a restart does not report a false fleet-wide outage. The window is measured from a required uptime sensor (see below). The low count is never held; it works the entire time.
- **A loud error if set up wrong.** If a required parameter is missing or misconfigured, or a threshold or margin is invalid, the sensor reports a `setup_error` state and does nothing until you fix it, rather than failing quietly. An `ok` attribute lets automations tell a working sensor from a broken one.
- **Scope by label, area, device, or entity**, with include and exclude, and exclude has priority.

The full design, the reasoning behind each choice, and worked examples are in the article: <https://xeazy.com/battery-entity-sentinel-blueprints/>

## Before You Start: the Uptime Requirement

Battery Sentinel measures its startup grace from a Home Assistant **uptime sensor in timestamp mode**, so it needs one in place before it will work.

The startup grace needs to know how long the system has been up. The obvious way to track that, reading the sensor's own boot time from its attributes, is unreliable: Home Assistant has a long-standing bug ([#115585](https://github.com/home-assistant/core/issues/115585)) where a trigger-based template sensor sometimes cannot read its own attributes during evaluation. The uptime sensor sidesteps it: an independent entity, read fresh every evaluation, reliable across restarts and reloads.

Most systems already have it. If yours does not, add the **[Uptime Integration](https://www.home-assistant.io/integrations/uptime/)** (Settings, Devices & Services, Add Integration, Uptime). It creates `sensor.uptime`, whose state is a timestamp like `2026-06-26T14:56:59+00:00`. Confirm it is in **timestamp** mode, not a number of days.

If the uptime sensor is missing, unavailable, or not a timestamp, Battery Sentinel reports a `setup_error` state with a message naming the problem, watches nothing, and starts working the moment you fix it.

## Import the Blueprint

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Ftemplate%2Fbattery_sentinel.yaml)

Or paste this URL into Settings, Automations & Scenes, Blueprints, Import Blueprint:

```
https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/template/battery_sentinel.yaml
```

Importing only registers a blueprint. It does not create a sensor. After import, the file lands at `config/blueprints/template/TheThinkingHome/battery_sentinel.yaml`. Verify it is there, you will need that path next.

## Setting Up the Sensor

### The Catch with Template Blueprints

Automation blueprints get a Create automation button on the Blueprints page. Template blueprints do not, and that is expected, not a fault. They become real entities through a short piece of YAML called `use_blueprint`, which takes a path and a set of inputs.

### What a Package Is, and How to Create One

A package is a single YAML file holding a bundle of related configurations that Home Assistant folds in at startup. Packages are optional, but they keep related things together.

If you have never used packages, switch them on once. In `configuration.yaml`, add the `packages:` line (under your existing `homeassistant:` section if you have one):

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Then create a folder named `packages` next to your `configuration.yaml`. Every YAML file you drop in there is a package.

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
        scan_interval: "/1"
        sensor_name: Battery Sentinel
        unique_id: battery_sentinel
```

To narrow the sensor, add an `include_target` and an `exclude_target`. If you also build Entity Sentinel, put both `use_blueprint` blocks in the same package file, each with its own `unique_id`.

### Load It

A brand-new package file needs a full Home Assistant restart to register the first time. After that, adding more blocks to the same file only needs a reload through Developer Tools, YAML, Reload Template Entities.

When it comes up you will have a sensor named after `sensor_name`. Watch it in Developer Tools, States. The state is the count; the `devices` attribute lists the flagged devices.

## Parameters

| Input | Required | Default | Accepted values | What it does |
| --- | --- | --- | --- | --- |
| `unique_id` | Yes | none | any short text id, distinct per sensor | Registers the entity so you can rename it or place it in an area. Reusing one collides two sensors. |
| `uptime_sensor` | Yes | `sensor.uptime` | an uptime sensor in timestamp mode | The clock the startup grace is measured from. Missing or non-timestamp yields `setup_error`. See "Before You Start." |
| `low_threshold` | No | `20` | 0 to 100 | A battery percentage at or below this counts as low. Binary sensors reading `on` always count. Out of range yields `setup_error`. |
| `low_clear_margin` | No | `2` | 0 or greater | How far above the threshold a flagged device must climb before it clears. The hysteresis band. Negative yields `setup_error`. |
| `include_target` | No | empty | entities, areas, devices, or **labels** | Which devices to watch. Empty watches every battery device-class entity. Areas and devices are filtered to batteries; labels expand on entity, device, or area. |
| `exclude_target` | No | empty | entities, areas, devices, or labels | Which to leave out. Exclude always wins over include. |
| `scan_interval` | No | `/1` | `/1`, `/2`, `/3`, `/4`, `/6`, `/8`, `/12`, `/24` | How often the sensor re-scans, in hours. `/1` is hourly, `/2` every two hours, `/6` four times a day. Battery levels drift slowly, so hourly or slower is plenty. |
| `startup_grace_seconds` | No | `240` | 0 or greater (seconds) | How long after Home Assistant starts to hold the unavailable list empty while the mesh repopulates, measured from the uptime sensor. |
| `refresh_button` | No | `input_button.none` | an `input_button` | Optional. Pressing it re-evaluates immediately. Point several Sentinels at one button to refresh them together. It re-scans; it cannot force a device to report. |
| `sensor_name` | No | `Battery Sentinel` | any text | The friendly name, and what the entity id is built from. |
| `debug_enabled` | No | `false` | on or off | When on, writes one diagnostic line to the system log each evaluation, carrying the version, state, and both the low and unavailable lists. |

## Setting the Grace Period

`startup_grace_seconds` is an input to set carefully because the right value depends on your hardware. Measured from the uptime sensor (the moment the Home Assistant process started), it has to cover your hub's full startup: boot time, plus the time your Zigbee or Z-Wave mesh takes to re-route and report in. This cannot be guessed, only deliberately timed.

The default `240` (four minutes) suits a fast mini-PC with a stable mesh; a slower hub with a large Zigbee or Z-Wave network may need five or six. Set it too short and a restart can flash a brief false outage before the mesh finishes; set it generously and the only cost is that a genuine outage in the first few minutes after a restart waits until the window closes to show.

## Scoping: Choosing What the Sensor Watches

The sensor decides what to watch through `include_target` and `exclude_target`. Each accepts entities, areas, devices, or **labels**.

**Default, watch everything.** Leave `include_target` out and the sensor watches every battery device-class entity in your system. This catches phones, tablets, and watches too; those charge nightly and are rarely what you want, which is what `exclude_target` is for.

**Include by entity** to watch exactly the entities you name. An explicit list is trusted as given.

**Exclude always wins.** `exclude_target` removes its targets from whatever the include produced, so it is how you drop the always-charged devices from a wide scope.

**Area or device** scopes are swept and filtered to the `device_class: battery` entities within them, dropping the rest (signal strength, connectivity, and so on).

**Label** scopes resolve on all three attachment points. Home Assistant lets you apply a label to an entity, a device, or an area; Battery Sentinel reads all three, so a label on a device or a room expands to the batteries it covers, not just labels sitting directly on entities. This works on both sides: a device labeled to be ignored is reliably excluded, not silently watched anyway.

```yaml
        include_target:
          label_id:
            - battery_watch
        exclude_target:
          entity_id:
            - sensor.phone_battery_level
```

## What the Sensor Looks Like

With two cells below the threshold and one battery offline:

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

The state is the number you badge or trigger on, or the string `setup_error` if misconfigured. The attributes carry the detail:

- **`state`** the count of low batteries, or `setup_error`.
- **`ok`** `true` when the sensor is functioning, `false` only in `setup_error`. The safe field to gate automations on.
- **`error`** empty normally; names the bad input when in `setup_error`.
- **`uptime_status`** the parsed uptime timestamp, or the raw bad value when the uptime sensor is the problem.
- **`total_monitored`** how many battery entities fell within the scope.
- **`devices`** the low batteries, each with `name`, `entity_id`, `area`, `level` (null for binary), and `battery_type` if exposed.
- **`unavailable_count`** / **`unavailable_entities`** the parallel list of in-scope batteries currently `unavailable`, `unknown`, or `missing`, held empty during the startup grace.
- **`boot_time`** the start time the grace is measured from, or null in `setup_error`.
- **`last_evaluated`** the timestamp of the most recent evaluation, a heartbeat you can watch to confirm the sensor is still running. Stored in UTC; see the timestamp note below.

### A Note on Timestamps

The timestamp attributes (`uptime_status`, `boot_time`, `last_evaluated`) are stored in **UTC**, in ISO-8601 form with the offset, for example `2026-06-28T23:26:05+00:00`. That is the portable, unambiguous form, and it is what an automation, a dashboard, or a companion blueprint reads. To show one in local time on a dashboard, convert it, for example `{{ state_attr('sensor.battery_sentinel', 'last_evaluated') | as_datetime | as_local }}`.

The optional **debug log line** is the one exception. As of 1.0.2-beta it prints its timestamps in your system's local time, since the log is for a person reading it, not for a machine. So a time in the log will not match the same field read raw from the attribute: the log is local, the attribute is UTC, and both point at the same instant. This split is deliberate, and it is why a quick mental subtraction of a log time against a UTC attribute can look off by your timezone offset.

### The `setup_error` State

If the sensor cannot run correctly it sets its state to `setup_error`, sets `ok` to false, and explains the problem in `error`. It watches nothing until fixed, then recovers on its own. Two things trigger it: the uptime sensor (missing, unavailable, or not a timestamp), or an invalid input, `low_threshold` outside 0 to 100, a negative `low_clear_margin`, or a negative `startup_grace_seconds`. The import UI constrains these, so you will mostly meet `setup_error` only via a `use_blueprint` typo or an unconfigured uptime sensor.

## A Note on Hysteresis After a Restart

The hysteresis band remembers which batteries were low on the previous evaluation, and that memory is read from the sensor's own prior output, the one piece of self-referential state with no external source. In the rare event the [#115585](https://github.com/home-assistant/core/issues/115585) bug strikes on a given evaluation, that memory reads empty for one tick and a battery sitting inside the clear margin briefly drops off the list, returning on the next clean evaluation. This is a known, self-correcting limitation; it cannot be removed without giving up hysteresis. The reasoning is in the article.

## Putting the Signal to Work

A dashboard card listing what is low, with the battery type:

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

A gate that holds an automation when the list is not clean. The `ok` check comes first, so a misconfigured sensor does not wave the automation through:

```yaml
condition:
  - condition: template
    value_template: >-
      {{ is_state_attr('sensor.battery_sentinel', 'ok', true)
         and states('sensor.battery_sentinel') | int(0) == 0 }}
```

Do not gate on `int()` math alone: `int('setup_error')` evaluates to 0, which would read a broken sensor as "all clear." Always check `ok` first.

More worked examples are in the article: <https://xeazy.com/battery-entity-sentinel-blueprints/>

## Now What? 

You have a sensor that always knows which batteries are low. The question is what reads it, and that list is open by design.

The first consumer is built and ready: **[Sentinel Notify](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/automation/sentinel_notify.md)**, a companion automation blueprint that turns this sensor into change-aware notifications, telling you the moment a battery drops below the threshold and again when you replace it, by name and with its type, without nagging you in between. It is a separate blueprint in this same repository.

Beyond notifying, the same `devices` attribute can drive anything. Two companions are on the drawing board, named here because they came up often enough to be worth building:

A **shopping-list companion** that keeps a to-do list in sync with the batteries currently low, one checkable item per battery with its type, so the right battery is bought before the swap and ticked off once it is. Because the sensor already knows what is low, this is a clean reader: it adds what is new, skips what is already listed, and can clear an item when the battery is replaced.

A **logging companion** that records a timestamped history as batteries go low and recover, so "this device has needed a new battery three times this year" becomes a record you can look back on rather than a thing you half-remember.

Neither is built yet. If there is enough interest, I will build them, or you can build them yourself. Reading a Sentinel sensor is straightforward, and its attributes are all documented above, so a consumer that does exactly what you want is well within reach. That is the whole point of a sensor over a sealed automation: the list of what can consume it is never closed, and the next useful reader is whatever you need.

## Changelog

| Version | Notes |
| --- | --- |
| 1.0.2-beta | Debug log line now prints its timestamps (`boot_time` and `last_evaluated`) in local time for readability. Display only: the stored attributes are unchanged and remain ISO-8601 UTC, the portable form anything downstream reads. |
| 1.0.1-beta | Re-evaluation cadence moved to hours, the production setting (minutes was a development affordance for fast testing). The `scan_interval` selector now offers 1, 2, 3, 4, 6, 8, 12, and 24 hours, defaulting to hourly, and the sensor fires at the top of the chosen hour. |
| 1.0.0-beta | First beta. Counts batteries at or below a threshold with a hysteresis deadband, handles percentage and binary battery entities, reports unavailable and missing batteries in a parallel `unavailable_entities` attribute. Scope by entity, label, area, or device with include and exclude (exclude wins; labels expand to entities, devices, and areas on both sides). Startup grace measured from a required uptime sensor in timestamp mode. Loud `setup_error` state with `error` and `uptime_status` attributes when the uptime sensor is missing or not a timestamp, or when `low_threshold`, `low_clear_margin`, or `startup_grace_seconds` is invalid; the sensor fails safe and self-heals. Added an `ok` boolean so automations can gate on it rather than doing integer math on a state that may read `setup_error`. Timezone-safe grace math. Debug log carries the version, state, uptime status, and both arrays. |

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>). This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.
