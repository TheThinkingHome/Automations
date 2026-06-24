# Battery Sentinel (Alpha)

*One sensor that watches every battery device in the house, for the ones running low and the ones that quietly stopped reporting.*

A free blueprint from **The Thinking Home** at [xeazy.com](https://xeazy.com).

This is the 1.0.0-alpha build under active testing. The design may still change. Do not rely on it for anything load-bearing until it graduates.

---

## What it is

Battery Sentinel is a template blueprint. It creates one trigger-based sensor whose state is a count of battery devices that need attention, and whose attributes list those devices with their area, level, and battery type. The sensor does no notifying on its own. It is a signal. Notifications, dashboard cards, to-do entries, and any other automation read from it.

This is the deliberate split. The sensor reports the state of the fleet. What you do about it is a separate automation you write, or one you import alongside this. Sophisticated setups build whatever they want on top of a clean signal instead of being locked into one notification format.

A companion automation blueprint, **Battery Sentinel Companion**, is coming. It will read this sensor and turn it into notifications and to-do entries for anyone who wants a turnkey consumer rather than writing their own. Until then, the consumption examples below show how to read the sensor directly.

## What it does that the common battery blueprints do not

It finds two different failures, not one.

**Low.** A battery percentage at or below your threshold, or a binary battery sensor reading on. A value-hysteresis band stops a device hovering at the threshold from flapping in and out of the list. A device enters the low list when it drops below the threshold and only clears when it climbs back above the threshold plus a small margin. Because batteries do not recharge themselves, the only thing that clears a flagged device is a fresh cell, which is exactly the behavior you want.

**Silent.** A device that has stopped reporting. This is the failure a percentage check cannot see. A battery does not always fade gracefully from twenty percent to ten to five. Sometimes it sits reading eighty percent for days because eighty was the last number it ever sent, while the device is actually dead and Home Assistant never marked it unavailable because nothing told it to. A plain unavailable check sails right past this. Battery Sentinel catches it by asking a different question: did this device report at all within the window, regardless of what number it is still showing.

The silent check is judged once a day at an hour you choose, not continuously. That distinction matters. Many battery devices, water leak sensors especially, legitimately report only once or twice a day. A rolling twenty-four hour window would flag those every day they run a little late. Judging silence once a day against a configurable lookback lets a once-a-day reporter stay quiet while a genuinely dead device is caught.

## Requirements

Home Assistant 2026.5.4 or newer. No custom integrations are required. If you run the Battery Notes integration, the sensor will pick up battery types automatically where they are exposed; if you do not, the battery type field is simply blank.

## Install

Import the blueprint, then build a sensor from it.

1. **Import.** Settings, Automations & scenes, Blueprints, Import blueprint, paste the blueprint URL, Preview, Import.
2. **Create the sensor.** On the Battery Sentinel blueprint, select Create automation, fill in the inputs, give it a unique id, and save.

Build it more than once if you want separate single-purpose sensors. A common pattern is one sensor in both mode for a dashboard badge, or a low sensor and a silent sensor kept apart so each can drive its own automation.

## Inputs

**Monitor mode.** What the state counts. Low only, silent only, or both.

**Low battery threshold.** A percentage at or below this counts as low. Default 20.

**Hysteresis clear margin.** How far above the threshold a flagged device must climb before it clears. Default 2, so a device flagged at the 20 threshold clears only above 22.

**Daily check hour.** The hour the silent check is judged, 0 to 23. Default 8.

**Silent lookback window.** How many hours back from the check hour a device must have reported to count as alive. Default 24. Raise it if normal daily reporters keep getting flagged. A leak sensor that drifts can be given 28, which looks back from 8 AM today to 4 AM yesterday.

**Include.** Devices, areas, entities, or labels to watch. Leave empty to watch every battery device-class entity automatically. Set a label here to watch only the devices you tag, which is the clean answer to a house with thousands of auto-added entities.

**Exclude.** Devices, areas, entities, or labels to leave out. Exclude always wins over include. Use it for phones, tablets, or a device you are deliberately running down.

**Re-evaluation interval.** How often the sensor re-scans, from every hour to every twelve hours. Default every 2 hours, which is light and plenty for batteries.

**Sensor name and unique id.** The name and registry id for the sensor this creates.

## What the sensor looks like

On a day with two low cells and one device gone quiet, in both mode:

```yaml
sensor.battery_sentinel
State: 3

monitor_mode: both
total_monitored: 27
last_evaluated: '2026-06-24T10:15:00-05:00'
low_count: 2
silent_count: 1
low:
  - name: Master Bath Motion
    entity_id: sensor.master_bath_motion_battery
    area: Master Bath
    level: 12
    battery_type: CR2450
  - name: Front Gate Contact
    entity_id: sensor.front_gate_contact_battery
    area: Entry
    level: 18
    battery_type: AAA
silent:
  - name: Guest Room Temperature
    entity_id: sensor.guest_room_temp_battery
    area: Guest Bedroom
    last_seen: '2026-06-21T03:14:08-05:00'
    age: 3 days
    battery_type: CR2032
silent_eval_date: '2026-06-24'
```

The state is the single number you badge or trigger on. The `low` and `silent` lists are structured objects, not a pre-joined string, so each consumer formats them its own way. This is the specific thing that broke older battery blueprints: they built the message text inside the template, and the empty-list and truncation bugs followed. Here the sensor reports data and the consumer writes the words.

## Consuming the sensor

A notification automation reading the both-mode sensor:

```yaml
trigger:
  - trigger: numeric_state
    entity_id: sensor.battery_sentinel
    above: 0
action:
  - action: notify.mobile_app_yourphone
    data:
      title: Battery attention needed
      message: >-
        {% set a = state_attr('sensor.battery_sentinel', 'low') %}
        {% set s = state_attr('sensor.battery_sentinel', 'silent') %}
        {% if a %}Low: {{ a | map(attribute='name') | join(', ') }}.{% endif %}
        {% if s %}Silent: {{ s | map(attribute='name') | join(', ') }}.{% endif %}
```

A markdown dashboard card listing the low devices with their area and battery type:

```yaml
type: markdown
content: >-
  {% for d in state_attr('sensor.battery_sentinel', 'low') %}
  - **{{ d.name }}** ({{ d.area }}) {{ d.level }}%, {{ d.battery_type }}
  {% endfor %}
```

Because the data is structured, the same sensor can feed a to-do list, a REST export, a voice summary, or a template that only alerts when the silent list is non-empty. The notification is just the least of what the sensor can drive.

## Known alpha limitations

These are the things still in flight. They are listed honestly because this is an alpha.

- **Voltage sensors are not yet handled.** Devices that report battery as a voltage rather than a percentage are skipped for now. Percentage and binary support comes first, voltage is the next layer.
- **Attributes are uniform across modes.** A low-mode sensor still carries `silent_count` and `silent` keys, set to zero and empty. Per-mode key omission fights Home Assistant's static attribute schema, so for the alpha the keys are constant and the `monitor_mode` attribute tells you what the sensor is actually watching.
- **The re-evaluation interval is hour-based.** Sub-hour cadences like every fifteen minutes are not in this build. Hourly is the floor for now.
- **The silent list is frozen between daily checks.** Because silence is judged once a day, the `silent` list and its age strings hold yesterday's values until the next daily check. This is intended, but it means the age figures are a snapshot, not live.
- **Battery type is best-effort.** It is read from a `battery_type` attribute where present. Deeper Battery Notes support is a later enhancement.

## Changelog

- **1.0.0-alpha.** Initial build. Low and silent detection in one sensor. Value hysteresis on the low check, so a device hovering at the threshold does not flap. Once-a-day silent check that spares legitimate daily reporters. Include and exclude accepting entities, areas, devices, or labels, with exclude winning. Low, silent, and both monitor modes. Percentage and binary battery sensors supported. Voltage staged for a later build. Increments to 1.0.0-beta, then 1.0.0, once it runs complete on a live system.

## License

GPL-3.0-or-later. Copyright (C) 2026 James Lander, The Thinking Home ([xeazy.com](https://xeazy.com)).
